/*
**
** Copyright (C) 2008, The Android Open Source Project
**
** Licensed under the Apache License, Version 2.0 (the "License");
** you may not use this file except in compliance with the License.
** You may obtain a copy of the License at
**
**     http://www.apache.org/licenses/LICENSE-2.0
**
** Unless required by applicable law or agreed to in writing, software
** distributed under the License is distributed on an "AS IS" BASIS,
** WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
** See the License for the specific language governing permissions and
** limitations under the License.
*/

//#define LOG_NDEBUG 0
#define LOG_TAG "Camera"
#include <utils/Log.h>
#include <utils/String16.h>

#include <camera/Camera.h>
#include <android/hardware/ICameraService.h>
#include <camera/android/hardware/ICamera.h>
#include <camera/IMetadataCallbacks.h>

namespace android {

using binder::Status;

status_t Camera::setMetadataCallback(sp<IMetadataCallbacks>& cb)
{
    ALOGV("Camera::setMetadataCallback");
    sp <::android::hardware::ICamera> c = mCamera;
    if (c == 0) {
        ALOGE("Camera::setMetadataCallback mCamera=NULL");
        return NO_INIT;
    }
    if (cb == NULL) {
        ALOGE("Camera::setMetadataCallback cb=NULL");
        return NO_INIT;
    }
    status_t res = OK;
    res = c->setMetadataCallback(cb);

    return res;
}

//!++
#if 1   // defined(MTK_CAMERA_BSP_SUPPORT)

status_t
Camera::
getProperty(String8 const& key, String8& value)
{
    const sp<::android::hardware::ICameraService>& cs = getCameraService();
    if (cs == 0) {
        ALOGE("Camera::getProperty cannot get CameraService");
        return UNKNOWN_ERROR;
    }

    String16 s16Key(key);
    String16 s16Value;
    Status res = cs->getProperty(s16Key, &s16Value);
    if ( !res.isOk() ) {
        ALOGE("Camera::getProperty Error");
        return UNKNOWN_ERROR;
    }
    value = String8(const_cast<const String16&>(s16Value));
    return OK;
}


status_t
Camera::
setProperty(String8 const& key, String8 const& value)
{
    const sp<::android::hardware::ICameraService>& cs = getCameraService();
    if (cs == 0) {
        ALOGE("Camera::getProperty cannot get CameraService");
        return UNKNOWN_ERROR;
    }

    String16 s16Key(key);
    String16 s16Value(value);
    Status res = cs->setProperty(s16Key, s16Value);
    if ( !res.isOk() ) {
        ALOGE("Camera::setProperty Error");
        return UNKNOWN_ERROR;
    }
    return OK;
}

#endif
//!--

}; // namespace android
