#ifndef ANDROID_MTK_HWC_H
#define ANDROID_MTK_HWC_H

#include <vector>
#include <map>
#include <memory>
#include <set>

#include <utils/Singleton.h>
#include <utils/KeyedVector.h>
#include <cutils/native_handle.h>
#include <hardware/hwcomposer.h>
#include <hwc_priv.h>

namespace android {

class IBinder;
class DisplayDevice;
class SurfaceFlinger;

// The interface of HWC2 is changed to object base instead of in-out struct.
// HWC1 passes data of MTK features by the in-out struct in the past, but the way
// is broken now. In HWC2, SF passes data of MTK features by MtkHwc
class MtkHwc : public Singleton<MtkHwc>
{
public:
    MtkHwc();
    ~MtkHwc();

    void setFlinger(const sp<SurfaceFlinger>& flinger);

    void setHwc(hwc_composer_device_1_t* hwc);

    // called in SurfaceFlinger::setUpHWComposer()
    void createHwcPrepareData(const DefaultKeyedVector< wp<IBinder>, sp<DisplayDevice> >& displays);

    // called in HWComposer::prepare()
    void onPrepareHead(const size_t&, hwc_display_contents_1_t* []);

    // called in HWComposer::prepare()
    void onPrepareTail(const size_t&, hwc_display_contents_1_t* []);

    std::shared_ptr<HwcPrepareData> getHwcPrepareData();

    nsecs_t getRefreshTimestamp(const int32_t& dpy);

private:
    hwc_composer_device_1_t* mHwc;

    std::shared_ptr<HwcPrepareData> mHwcPrepareData;

    wp<SurfaceFlinger> mFlinger;
};

inline std::shared_ptr<HwcPrepareData> getHwcPrepareData() {
    return MtkHwc::getInstance().getHwcPrepareData();
}

inline std::shared_ptr<HwcPrepareData::DisplayData> getDisplayOfHwcPrepareData(const uint32_t& id) {
    if (getHwcPrepareData() == nullptr) {
        ALOGE("%s(%d): mHwcPrepareData is NULL!", __func__, __LINE__);
    }
    return MtkHwc::getInstance().getHwcPrepareData()->displays[id];
}

inline std::shared_ptr<HwcPrepareData::LayerData> getLayerOfHwcPrepareData(const uint32_t& sequence) {
    if (getHwcPrepareData() == nullptr) {
        ALOGE("%s(%d): mHwcPrepareData is NULL!", __func__, __LINE__);
    }
    return MtkHwc::getInstance().getHwcPrepareData()->layers[sequence];
}

}

#endif // ANDROID_MTK_HWC_H
