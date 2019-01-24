#include "MtkHwc.h"

#include <cutils/log.h>

#include <hardware/hardware.h>
#include <hardware/hwcomposer.h>
#include <hwc_priv.h>

#include "DisplayDevice.h"
#include "Layer.h"

#include "utils/CallStack.h"

using namespace android;
using namespace std;

// --------------------------------------------------------------------

HwcPrepareData::HwcPrepareData()
{
    displays.resize(32);
    for (size_t i = 0; i < displays.size(); ++i)
        displays[i] = nullptr;
}

HwcPrepareData::DisplayData::DisplayData(const uint32_t& tid)
    : id(tid)
    , flags(0)
{
}

HwcPrepareData::LayerData::LayerData(const buffer_handle_t& thnd)
    : hnd(thnd)
    , hints(0)
    , flags(0)
{
}

// --------------------------------------------------------------------
ANDROID_SINGLETON_STATIC_INSTANCE(MtkHwc);

MtkHwc::MtkHwc() {
}


void MtkHwc::setFlinger(const sp<SurfaceFlinger>& flinger) {
    if (flinger == nullptr) {
        ALOGE("%s(%d): flinger is NULL!", __func__, __LINE__);
    }
    mFlinger = flinger;
}

void MtkHwc::setHwc(hwc_composer_device_1_t* hwc) {
    if (hwc == nullptr) {
        ALOGE("%s(%d): hwc device is NULL!", __func__, __LINE__);
    }
    mHwc = hwc;
}

void MtkHwc::createHwcPrepareData(
    const DefaultKeyedVector< wp<IBinder>, sp<DisplayDevice> >& displays) {
    mHwcPrepareData = make_shared<HwcPrepareData>();
    if (mHwcPrepareData == nullptr) {
        ALOGE("%s(%d): createHwcPrepareData fail", __func__, __LINE__);
    }

    // Is mGeometryInvalid useful?
    for (size_t dpy = 0; dpy < displays.size(); dpy++) {
        sp<const DisplayDevice> displayDevice(displays[dpy]);
        if (displayDevice == nullptr) {
            continue;
        }
        const auto hwcId = displayDevice->getHwcDisplayId();

        if (hwcId >= 0) {
            auto&& display = make_shared<HwcPrepareData::DisplayData>(hwcId);
            if (display == nullptr) {
                ALOGE("%s(%d): create DisplayData fail", __func__, __LINE__);
                continue;
            }

            mHwcPrepareData->displays[hwcId] = display;
            const Vector<sp<Layer>>& currentLayers(
                    displayDevice->getVisibleLayersSortedByZ());
            for (auto& layer : currentLayers) {
                if (layer == nullptr) {
                    ALOGE("%s(%d): layer is null", __func__, __LINE__);
                    continue;
                }
                auto&& sequence = layer->sequence;
                auto&& activeBuffer = layer->getActiveBuffer();
                if (mHwcPrepareData->layers.find(sequence) == mHwcPrepareData->layers.end()) {
                    mHwcPrepareData->layers[sequence] = make_shared<HwcPrepareData::LayerData>(
                        activeBuffer == nullptr? nullptr : activeBuffer->handle);
                }
            }
        }
    }
}

shared_ptr<HwcPrepareData> MtkHwc::getHwcPrepareData() {
    if (mHwcPrepareData == nullptr) {
        ALOGE("%s: mHwcPrepareData is NULL!", __func__);
    }
    return mHwcPrepareData;
}

void MtkHwc::onPrepareHead(const size_t& numDisplays, hwc_display_contents_1_t* list[]) {
    if (mHwcPrepareData == nullptr) {
        ALOGE("%s: mHwcPrepareData is NULL!", __func__);
    }

    auto&& flinger = mFlinger.promote();
    if (flinger == nullptr) {
        ALOGE("%s(%d): flinger is NULL!", __func__, __LINE__);
        return ;
    }
    for (size_t i = 0; i < numDisplays; ++i) {
        auto&& display = list[i];
        if (display == nullptr || mHwcPrepareData->displays[i] == nullptr) {
            continue;
        }

        auto&& layers = flinger->getLayerSortedByZForHwcDisplay(i);
        display->flags |= mHwcPrepareData->displays[i]->flags;
        if (layers.size() == 0) {
            continue;
        }

        LOG_ALWAYS_FATAL_IF(layers.size() != (display->numHwLayers - 1), "layers.size():%zu, display->numHwLayers:%zu", layers.size(), display->numHwLayers);

        for (size_t j = 0; j < display->numHwLayers - 1; ++j) {
            auto&& hwLayer = display->hwLayers[j];
            int32_t sequence = layers[j]->sequence;
            if (mHwcPrepareData->layers.find(sequence) == mHwcPrepareData->layers.end()) {
                continue;
            }
            auto&& layerData = mHwcPrepareData->layers[sequence];

            LOG_ALWAYS_FATAL_IF(hwLayer.handle != layerData->hnd, "(%zu:%zu)hwLayer->handle != layerData->hnd", i, j);

            hwLayer.flags |= layerData->flags;
            hwLayer.hints |= layerData->hints;

            if ((layerData->flags & HWC_DIM_LAYER) != 0)
                hwLayer.flags &= ~HWC_SKIP_LAYER;
        }
    }
}

void MtkHwc::onPrepareTail(const size_t& numDisplays, hwc_display_contents_1_t* list[]) {
    if (mHwcPrepareData == nullptr) {
        ALOGE("%s(%d): mHwcPrepareData is NULL!", __func__, __LINE__);
    }
    auto&& flinger = mFlinger.promote();
    if (flinger == nullptr) {
        ALOGE("%s(%d): flinger is NULL!", __func__, __LINE__);
        return ;
    }
    for (size_t i = 0; i < numDisplays; ++i) {
        auto&& display = list[i];
        if (display == nullptr || mHwcPrepareData->displays[i] == nullptr) {
            continue;
        }

        auto&& layers = flinger->getLayerSortedByZForHwcDisplay(i);
        mHwcPrepareData->displays[i]->flags = display->flags;
        if (layers.size() == 0) {
            continue;
        }

        LOG_ALWAYS_FATAL_IF(layers.size() != (display->numHwLayers - 1), "layers.size():%zu display->numHwLayers:%zu", layers.size(), display->numHwLayers);

        for (size_t j = 0; j < display->numHwLayers - 1; ++j) {
            auto&& hwLayer = display->hwLayers[j];
            if (mHwcPrepareData->layers.find(layers[j]->sequence) == mHwcPrepareData->layers.end()) {
                continue;
            }
            auto&& layerData = mHwcPrepareData->layers[layers[j]->sequence];

            LOG_ALWAYS_FATAL_IF(hwLayer.handle != layerData->hnd, "(%zu:%zu)hwLayer->handle != layerData->hnd", i, j);

            layerData->flags = hwLayer.flags;
            layerData->hints = hwLayer.hints;
        }
    }
}

nsecs_t MtkHwc::getRefreshTimestamp(const int32_t& dpy) {
    auto&& flinger = mFlinger.promote();
    if (flinger == nullptr) {
        ALOGE("%s(%d): flinger is NULL!", __func__, __LINE__);
        return 0;
    }

#ifdef USE_HWC2
    return flinger->getHwComposer().getRefreshTimestamp(dpy);
#else
    return flinger->getHwComposer().getRefreshPeriod(dpy);
#endif
}
