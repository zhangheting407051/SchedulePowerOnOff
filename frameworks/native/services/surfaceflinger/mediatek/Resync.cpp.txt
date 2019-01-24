#define ATRACE_TAG ATRACE_TAG_GRAPHICS

// This is needed for stdint.h to define INT64_MAX in C++
#define __STDC_LIMIT_MACROS

#include <math.h>

#include <cutils/log.h>
#include <cutils/properties.h>

#include <utils/String8.h>
#include <utils/Thread.h>
#include <utils/Trace.h>
#include <utils/Vector.h>

#include "EventThread.h"
#include "SurfaceFlinger.h"
#include "mediatek/Resync.h"
#include "mediatek/MtkHwc.h"

#include "ged/ged.h"

#define ATRACE_NAME(name) android::ScopedTrace ___tracer(ATRACE_TAG, name)
#define ATRACE_BUFFER(x, ...)                                                   \
    if (ATRACE_ENABLED()) {                                                     \
        char ___traceBuf[256];                                                  \
        snprintf(___traceBuf, sizeof(___traceBuf), x, ##__VA_ARGS__);           \
        android::ScopedTrace ___bufTracer(ATRACE_TAG, ___traceBuf);             \
    }

namespace android {

#define VSYNC_DEFAULT_SETTING_FPS "persist.mtk.sf.fps.upper_bound"
#define WARNING_HW_VSYNC_ERROR 0.1f


// gHwPeriod is for storing the actual hw vsync period
nsecs_t gHwPeriod = 0;
nsecs_t gIdlePeriod = 0;
nsecs_t gLastSync = 0;
nsecs_t gPrevSync = 0;
nsecs_t gWarningThreshold = 0;
nsecs_t gSyncNum = 3;

VSyncSourceData::VSyncSourceData(nsecs_t offset,
                     VSyncSource* vss,
                     DispSync::Callback* cb,
                     int32_t skipCount,
                     bool delay)
    : mVss(vss)
    , mCb(cb)
    , mOffset(offset)
    , mPendingOffset(0)
    , mLastEventTime(0)
    , mDelayChange(delay)
    , mIsPendingOffset(false)
    , mSkip(skipCount)
    , mPendingSkip(0)
    , mSkipCount(0)
{
}


Resync::Resync(SurfaceFlinger* flinger, int dpy, nsecs_t offset)
    : mPeriod(0)
    , mPhase(0)
    , mHasPendingOffset(false)
    , mOffsetChange(false)
    , mFps(0)
{
    if (flinger != NULL) {
        gHwPeriod = MtkHwc::getInstance().getRefreshTimestamp(dpy);
        if (gHwPeriod == 0) {
            ALOGW("failed to get the correct refresh period");
            gHwPeriod = 16666666;
        }

        gWarningThreshold = gHwPeriod * WARNING_HW_VSYNC_ERROR;

        gIdlePeriod = gHwPeriod * gSyncNum;

        // measure the fps of primary display
        mFps = 1e9 / gHwPeriod;
        mFps = (mFps + 5) / 10 * 10;
    }

    // apply fps setting of user
    char value[PROPERTY_VALUE_MAX];
    property_get(VSYNC_DEFAULT_SETTING_FPS, value, "0");
    int32_t fps = atoi(value);
    mSkipCount = isMultipleOfFps(fps);
    mOffset = offset;
    ALOGI("Display_%d: fps[%d] skipCount[%d:%d] offset[%" PRId64 "]",
           dpy, mFps, fps, mSkipCount, offset);
#ifndef MTK_EMULATOR_SUPPORT
    mGed = reinterpret_cast<void*>(ged_create());
#endif
}

Resync::~Resync() {
#ifndef MTK_EMULATOR_SUPPORT
    ged_destroy(reinterpret_cast<GED_HANDLE>(mGed));
#endif
}

void Resync::updateModelLocked(nsecs_t period, nsecs_t phase) {
    mPeriod = period;
    mPhase = phase;
}

void Resync::updateSyncTimeLocked(int num, nsecs_t sync) {
    if (num >= gSyncNum || (sync - gLastSync) <= gHwPeriod * 2) {
        gLastSync = sync;
    }

    // Mark a tag, if the error of HW VSync period is more than threshold
    static bool preStatus = 0;
    if (num > 1) {
        nsecs_t error = sync - gPrevSync - gHwPeriod;
        error = (error >= 0) ? error : -error;
        bool needReport = error > gWarningThreshold;
        if (preStatus != needReport) {
            preStatus = needReport;
            ATRACE_INT("WARNING_HW_VSYNC", needReport ? 1 : 0);
        }
    }
    gPrevSync = sync;
}

/*
 * Record all EventListener in whole system.
 * Because DispSyncSource can remove itself from DispSync list when it does
 * not need VSyncSource, we need a way to hold a list for adjusing phase.
 */
status_t Resync::registerEventListener(nsecs_t offset,
                                       VSyncSource* vss,
                                       DispSync::Callback* cb,
                                       bool delay) {
    Mutex::Autolock lock(mMutexVssd);
    for (size_t i = 0; i < mVsyncSourceList.size(); i++) {
        if (mVsyncSourceList[i].mVss == vss) {
            return BAD_VALUE;
        }
    }

    struct VSyncSourceData vssd(offset, vss, cb, mSkipCount, delay);

    mVsyncSourceList.push(vssd);
    return NO_ERROR;
}

/*
 * The function name is a smoke screen, because GED want to cover up itself
 * behavior.
 * Change EventListener's phase at the right time.
 * We have to guarantee that the interval is more than a period between each
 * AP VSync and SF VSync, so SF VSync should delay a vsync to change its
 * phase.
 */
status_t Resync::checkVsyncAccuracy(const Vector<DupCallbackInvocation>& civ)
{
    Mutex::Autolock lock(mMutexState);

    // Change the phase of pended EventListener in previous turn
    if (mHasPendingOffset) {
        mHasPendingOffset = false;
        for (size_t i = 0; i < mVsyncSourceList.size(); i++) {
            struct VSyncSourceData& vssd = mVsyncSourceList.editItemAt(i);
            // It is SF VSync. Does it fire a callback in this time?
            if (vssd.mIsPendingOffset) {
                for (size_t j = 0; j < civ.size(); j++) {
                    // SF VSync fire a callback, so we can change it.
                    if (civ[j].mCallback == vssd.mCb) {
                        ALOGV("VSyncSource_%zu start to change delay offset to %" PRId64, i, vssd.mPendingOffset);
                        vssd.mIsPendingOffset = false;
                        vssd.mVss->setPhaseOffset(vssd.mPendingOffset);
                        break;
                    }
                }
            }
            mHasPendingOffset |= mVsyncSourceList[i].mIsPendingOffset;
        }
    }

    // Change the phase of EventListener whose pending flag does not set
    if (mOffsetChange) {
        mOffsetChange = false;
        for (size_t i = 0; i < mVsyncSourceList.size(); i++) {
            struct VSyncSourceData& vssd = mVsyncSourceList.editItemAt(i);
            nsecs_t offset;
            {
                Mutex::Autolock lock(mMutexVssd);
                offset = vssd.mOffset;
            }
            if (offset != mOffset || (vssd.mIsPendingOffset && vssd.mPendingOffset != mOffset)) {
                // Now, only SF's mDelayChange is true, so we should not
                // change its phase immediately.
                if (vssd.mDelayChange) {
                    mHasPendingOffset = true;
                    vssd.mPendingOffset = mOffset;
                    vssd.mIsPendingOffset = true;
                    ALOGV("VSyncSource_%zu delay offset change(%" PRId64 ")", i, mOffset);
                    continue;
                }
                ALOGV("VSyncSource_%zu change offset to %" PRId64, i, mOffset);
                vssd.mVss->setPhaseOffset(mOffset);
            }
        }
    }

    return NO_ERROR;
}

/*
 * Check the reason what this listener is added.
 */
status_t Resync::addEventListener(DupEventListener* listener, nsecs_t wakeupLatency)
{
    nsecs_t oldOffset = 0;
    size_t i;
    for (i = 0; i < mVsyncSourceList.size(); i++) {
        if (mVsyncSourceList[i].mCb == listener->mCallback.get()) {
            oldOffset = mVsyncSourceList[i].mOffset;
            break;
        }
    }

    if (i < mVsyncSourceList.size() && oldOffset != listener->mPhase) {
        // This cause of removing EventListener is changing phase, so we
        // have to guarantee that the interval of vsync is more than a
        // a period of vsync.
        listener->mLastEventTime = mVsyncSourceList[i].mLastEventTime + mPeriod / 2 + mPhase -
                                   wakeupLatency;
        Mutex::Autolock lock(mMutexVssd);
        mVsyncSourceList.editItemAt(i).mOffset = listener->mPhase;
    } else {
        // It just want to get a vsync event.
        listener->mLastEventTime = systemTime() - mPeriod / 2 + mPhase - wakeupLatency;
    }

    return NO_ERROR;
}

/*
 * Reserve the last event time for adjusting offset
 */
status_t Resync::removeEventListener(const DupEventListener &listener)
{
    for (size_t i = 0; i < mVsyncSourceList.size(); i++) {
        struct VSyncSourceData& vssd = mVsyncSourceList.editItemAt(i);
        if (vssd.mCb == listener.mCallback.get()) {
            vssd.mLastEventTime = listener.mLastEventTime;
            break;
        }
    }

    return NO_ERROR;
}

/*
 * The interface is for adjusting FPS of SW VSync.
 * We adjust FPS through dropping VSync.
 */
status_t Resync::adjustVsyncPeriod(int32_t fps)
{
    ATRACE_CALL();
    Mutex::Autolock lock(mMutexState);
    int32_t skip = isMultipleOfFps(fps);
    ALOGI("Adjust vsyc fps: old[%d] new[%d:%d]", mSkipCount, fps, skip);
    if (mSkipCount != skip) {
        ATRACE_BUFFER("new fps: %d", fps);
        mSkipCount = skip;
        Mutex::Autolock lock(mMutexVssd);
        for (size_t i = 0; i < mVsyncSourceList.size(); i++) {
            struct VSyncSourceData& vssd = mVsyncSourceList.editItemAt(i);
            if (vssd.mSkipCount == vssd.mSkip) {
                vssd.mSkipCount = skip;
            }
            vssd.mSkip = skip;
        }
    }

    return NO_ERROR;
}

/*
 * Notify the listener that its vsync arrive.
 * We also drop its vsync in here, then FPS can be slow down.
 */
status_t Resync::fireCallbackInvocations(const Vector<DupCallbackInvocation>& civ)
{

    checkVsyncAccuracy(civ);

    bool notifyGed = false;
    size_t sfIndex = 0;
    Vector<struct DupCallbackInvocation> fireCb;
    {
        Mutex::Autolock lock(mMutexVssd);
        for (size_t i = 0; i < civ.size(); i++) {
            size_t j;
            for (j = 0; j < mVsyncSourceList.size(); j++) {
                struct VSyncSourceData& vssd = mVsyncSourceList.editItemAt(j);
                if (vssd.mCb == civ[i].mCallback.get()) {
                    // The skip count is zero, so we can notify listener
                    if (vssd.mSkipCount == 0) {
                        fireCb.push(civ[i]);

                        // notify ged that we fire a SF VSync
                        if (vssd.mDelayChange == true) {
                            notifyGed = true;
                            sfIndex = i;
                        }
                    }
                    if (vssd.mSkipCount != 0) {
                        vssd.mSkipCount--;
                    } else if (vssd.mSkip) {
                        vssd.mSkipCount = vssd.mSkip;
                    }
                    break;
                }
            }
            // some DispSync callbacks do not register at list, like ZeroPhaseTracer
            if (j >= mVsyncSourceList.size())
            {
                civ[i].mCallback->onDispSyncEvent(civ[i].mEventTime);
            }
        }
    }

    for (size_t i = 0; i < fireCb.size(); i++)
    {
        fireCb[i].mCallback->onDispSyncEvent(fireCb[i].mEventTime);
    }
    if (notifyGed) {
        calculateEventDelayTime(civ[sfIndex].mEventTime);
    }

    return NO_ERROR;
}

/*
 * The interface is for adjust vsync offset.
 */
status_t Resync::adjustVsyncOffset(nsecs_t offset)
{
    ATRACE_CALL();
    Mutex::Autolock lock(mMutexState);
    ALOGI("Adjust vsync offset: old[%" PRId64 "] new[%" PRId64 "]", mOffset, offset);
    if (mOffset != offset) {
        ATRACE_BUFFER("new offset: %" PRId64, offset);
        mOffset = offset;
        mOffsetChange = true;
    }

    return NO_ERROR;
}

/*
 * This function name is smoke screen.
 * Notify GED that SF get a VSync.
 */
void Resync::calculateEventDelayTime(nsecs_t /*timestamp*/)
{
#ifndef MTK_EMULATOR_SUPPORT
{
    ATRACE_NAME("calculateDelayTime");
    ged_vsync_calibration(reinterpret_cast<GED_HANDLE>(mGed), 0, mPeriod * (mSkipCount + 1));
}
#endif
}

/*
 * Judge whether panel fps is multiple of input value
 * If it is multiple, return multiple minus one.
 * Otherwise return 0.
 */
int32_t Resync::isMultipleOfFps(int32_t fps)
{
    if (fps <= 0) return 0;

    int32_t isMultiple = !(mFps % fps);
    if (isMultiple)
    {
        isMultiple = mFps / fps - 1;
    }

    return isMultiple;
}

void Resync::dump(String8& result)
{
    Mutex::Autolock lock(mMutexState);
    result.appendFormat("[DispSyncInfo]\n");
    result.appendFormat("  Primary Display FPS: %d\n", mFps);
    result.appendFormat("  Skip: %d\n", mSkipCount);
    result.appendFormat("  Offset: %" PRId64 "\n", mOffset);

    result.appendFormat("  Total VSync source: %zu\n", mVsyncSourceList.size());
    for (size_t i = 0; i < mVsyncSourceList.size(); i++)
    {
        result.appendFormat("[VSyc_Source_%zu_%s]\n", i, mVsyncSourceList[i].mDelayChange ? "SF" : "AP");
        result.appendFormat("  Offset: %" PRId64 "\n", mVsyncSourceList[i].mOffset);
        if (mVsyncSourceList[i].mIsPendingOffset)
        {
            result.appendFormat("  Pending Offset: %" PRId64 "\n", mVsyncSourceList[i].mPendingOffset);
        }
        else
        {
            result.appendFormat("  Pending Offset: N/A\n");
        }
        result.appendFormat("  SC: %d/%d\n", mVsyncSourceList[i].mSkipCount, mVsyncSourceList[i].mSkip);
    }
}

nsecs_t Resync::getVsyncOffset() const
{
    Mutex::Autolock lock(mMutexState);
    return mOffset;
}

} // namespace android
