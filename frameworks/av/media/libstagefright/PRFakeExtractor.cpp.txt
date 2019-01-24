/*
* Copyright (C) 2014 MediaTek Inc.
* Modification based on code covered by the mentioned copyright
* and/or permission notice(s).
*/
/*
 * Copyright (C) 2010 The Android Open Source Project
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
/*****************************************************************************
 *
 * Filename:
 * ---------
 *   PRFakeExtractor.cpp
 *
 * Project:
 * --------
 *   Everest
 *
 * Description:
 * ------------
 *   PlayReady SVP for 4k SVP
 *
 * Note:
 *   Condition to use the extractor:
 *      1, Should special video, mp4 with special head + h264
 *      2, Should setprop playready.fake.mode 1
 *      3, Should open FO MTK_PLAYREADY_SUPPORT && TRUSTONIC_TEE_SUPPORT &&
 *         MTK_SEC_VIDEO_PATH_SUPPORT
 *      4, local option control: MTK_PLAYREADY_FAKEMODE, in Android.mk,
 *         mark it to disable it
 * Author:
 * -------
 *   Xingyu Zhou (mtk80781)
 *
 ****************************************************************************/
#ifdef MTK_PLAYREADY_FAKEMODE
#define LOG_TAG "PRFakeExtractor"
#include "include/PRFakeExtractor.h"

#include <arpa/inet.h>
#include <utils/String8.h>
#include <media/stagefright/foundation/ADebug.h>
#include <media/stagefright/Utils.h>
#include <media/stagefright/DataSource.h>
#include <media/stagefright/MediaSource.h>
#include <media/stagefright/MediaDefs.h>
#include <media/stagefright/MetaData.h>
#include <media/stagefright/MediaErrors.h>
#include <media/stagefright/MediaBuffer.h>

#include <utils/Errors.h>
#include <media/stagefright/MediaBufferGroup.h>
#include <cutils/properties.h>

#include <fcntl.h>
#include <linux/ion.h>
#include <linux/ion_drv.h>
#include <ion/ion.h>
#include <unistd.h>
#include <linux/mtk_ion.h>
#include <ion.h>
namespace android {
static bool isFakeMode() {
    char value[PROPERTY_VALUE_MAX];
    if (property_get("playready.fake.mode", value, "0")) {
        bool _res = atoi(value);
        if (_res) {
            return true;
        }
    }
    return false;
}

class PRFakeSource : public MediaSource {
public:
    PRFakeSource(const sp<IMediaSource> &mediaSource);

    virtual status_t start(MetaData *params = NULL);
    virtual status_t stop();
    virtual sp<MetaData> getFormat();
    virtual status_t read(
            MediaBuffer **buffer, const ReadOptions *options = NULL);

    virtual status_t setBuffers(const Vector<MediaBuffer *> &buffers);

protected:
    virtual ~PRFakeSource();

private:
    sp<IMediaSource> mOriginalMediaSource;
    size_t mTrackId;
    mutable Mutex mPRFakeLock;
    size_t mNALLengthSize;
    bool mWantsNALFragments;

    struct CodecSpecificData {
        size_t mSize;
        uint8_t mData[1];
    };
    Vector<CodecSpecificData *> mCodecSpecificData;
    size_t mCodecSpecificDataIndex;
    status_t parseAVCCodecSpecificData(
            const void *data, size_t size,
            unsigned *profile, unsigned *level);
    status_t parseHVCCodecSpecificData(
            const void *data, size_t size,
            unsigned *profile, unsigned *level);
    void addCodecSpecificData(const void *data, size_t size);
    void clearCodecSpecificData();
    ReadOptions mOptions;

    struct VaMapStruct {
        size_t len;
        void *va;
        int ion_fd;
        int share_fd;
        ion_user_handle_t handle;
    };
    KeyedVector<void*, struct VaMapStruct> mPaMap;

    status_t secureCopy(void *src, size_t len, bool addPrefix = false);

    MediaBufferGroup *mGroup;
    MediaBuffer *mBuffer;
    PRFakeSource(const PRFakeSource &);
    PRFakeSource &operator=(const PRFakeSource &);
};

////////////////////////////////////////////////////////////////////////////////

PRFakeSource::PRFakeSource(const sp<IMediaSource> &mediaSource)
    : mOriginalMediaSource(mediaSource),
    mNALLengthSize(0),
    mWantsNALFragments(false) {
    mBuffer = NULL;
    mGroup = NULL;
    mCodecSpecificDataIndex = 0;

    const char *mime;
    bool success = getFormat()->findCString(kKeyMIMEType, &mime);
    CHECK(success);

    if (!strcasecmp(mime, MEDIA_MIMETYPE_VIDEO_AVC)) {
        uint32_t type;
        const void *data;
        size_t size;
        CHECK(getFormat()->findData(kKeyAVCC, &type, &data, &size));

        const uint8_t *ptr = (const uint8_t *)data;

        CHECK(size >= 7);
        CHECK_EQ((unsigned)ptr[0], 1u);  // configurationVersion == 1

        // The number of bytes used to encode the length of a NAL unit.
        mNALLengthSize = 1 + (ptr[4] & 3);
        unsigned profile, level;
        status_t err;
        if ((err = parseAVCCodecSpecificData(
                        data, size, &profile, &level)) != OK) {
            ALOGE("Malformed AVC codec specific data.");
        }
        getFormat()->remove(kKeyAVCC);
    } else if (!strcasecmp(mime, MEDIA_MIMETYPE_VIDEO_HEVC)) {

        uint32_t type;
        const void *data;
        size_t size;
        CHECK(getFormat()->findData(kKeyHVCC, &type, &data, &size));

        unsigned profile, level;
        status_t err;
        if ((err = parseHVCCodecSpecificData(
                        data, size, &profile, &level)) != OK) {
            ALOGE("Malformed HVCC codec specific data.");
        }
        getFormat()->remove(kKeyHVCC);
    }

    mOptions.clearSeekTo();
}

status_t PRFakeSource::parseHVCCodecSpecificData(
        const void *data, size_t size,
        unsigned *profile, unsigned *level) {
        const uint8_t *ptr = (const uint8_t *)data;
        if (size < 23) {
            ALOGE("b/23680780");
            return BAD_VALUE;
        }

        *profile = ptr[1] & 31;
        *level  = ptr[12];
        ptr += 22;
        size -= 22;


        size_t numofArrays = (char)ptr[0];
        ptr += 1;
        size -= 1;
        size_t j = 0, i = 0;

        for (i = 0; i < numofArrays; i++) {
            if (size < 3) {
                ALOGE("b/23680780");
                return BAD_VALUE;
            }
            ptr += 1;
            size -= 1;

            //Num of nals
            size_t numofNals = U16_AT(ptr);

            ptr += 2;
            size -= 2;

            for (j = 0; j < numofNals; j++) {
                if (size < 2) {
                    ALOGE("b/23680780");
                    return BAD_VALUE;
                }
                size_t length = U16_AT(ptr);

                ptr += 2;
                size -= 2;

                if (size < length) {
                    return BAD_VALUE;
                }
                addCodecSpecificData(ptr, length);

                ptr += length;
                size -= length;
            }
        }

    return OK;
}

status_t PRFakeSource::parseAVCCodecSpecificData(
        const void *data, size_t size,
        unsigned *profile, unsigned *level) {
    const uint8_t *ptr = (const uint8_t *)data;

    // verify minimum size and configurationVersion == 1.
    if (size < 7 || ptr[0] != 1) {
        return ERROR_MALFORMED;
    }

    *profile = ptr[1];
    *level = ptr[3];

    // There is decodable content out there that fails the following
    // assertion, let's be lenient for now...
    // CHECK((ptr[4] >> 2) == 0x3f);  // reserved

    // size_t lengthSize = 1 + (ptr[4] & 3);

    // commented out check below as H264_QVGA_500_NO_AUDIO.3gp
    // violates it...
    // CHECK((ptr[5] >> 5) == 7);  // reserved

    size_t numSeqParameterSets = ptr[5] & 31;

    ptr += 6;
    size -= 6;

    for (size_t i = 0; i < numSeqParameterSets; ++i) {
        if (size < 2) {
            return ERROR_MALFORMED;
        }

        size_t length = U16_AT(ptr);

        ptr += 2;
        size -= 2;

        if (size < length) {
            return ERROR_MALFORMED;
        }

        addCodecSpecificData(ptr, length);

        ptr += length;
        size -= length;
    }

    if (size < 1) {
        return ERROR_MALFORMED;
    }

    size_t numPictureParameterSets = *ptr;
    ++ptr;
    --size;

    for (size_t i = 0; i < numPictureParameterSets; ++i) {
        if (size < 2) {
            return ERROR_MALFORMED;
        }

        size_t length = U16_AT(ptr);

        ptr += 2;
        size -= 2;

        if (size < length) {
            return ERROR_MALFORMED;
        }

        addCodecSpecificData(ptr, length);

        ptr += length;
        size -= length;
    }

    return OK;
}

void PRFakeSource::addCodecSpecificData(const void *data, size_t size) {
    CodecSpecificData *specific =
        (CodecSpecificData *)malloc(sizeof(CodecSpecificData) + size - 1);

    specific->mSize = size;
    memcpy(specific->mData, data, size);

    mCodecSpecificData.push(specific);
}

void PRFakeSource::clearCodecSpecificData() {
    for (size_t i = 0; i < mCodecSpecificData.size(); ++i) {
        free(mCodecSpecificData.editItemAt(i));
    }
    mCodecSpecificData.clear();
    mCodecSpecificDataIndex = 0;
}

PRFakeSource::~PRFakeSource() {
    Mutex::Autolock autoLock(mPRFakeLock);
    clearCodecSpecificData();

    for (size_t i=0; i<mPaMap.size(); ++i) {
        ALOGI("munmap :%p", mPaMap[i].va);
        ion_munmap(mPaMap[i].ion_fd, mPaMap[i].va, mPaMap[i].len);

        // do not close share fd, due to omx would close it
        // Extractor and omx in one process, result in share fd is the same
        // ion_share_close(mPaMap[i].ion_fd, mPaMap[i].share_fd);
        ion_free(mPaMap[i].ion_fd, mPaMap[i].handle);
        close(mPaMap[i].ion_fd);
    }

    if (mBuffer != NULL) {
        mBuffer->release();
        mBuffer = NULL;
    }
    if (mGroup != NULL) {
        delete mGroup;
        mGroup = NULL;
    }
}

status_t PRFakeSource::start(MetaData *params) {
    // svp h264 need mWantsNALFragments false
    mWantsNALFragments = false;
    ALOGI("SVP do not use nal fragments");
   return mOriginalMediaSource->start(params);
}

status_t PRFakeSource::stop() {
    return mOriginalMediaSource->stop();
}

sp<MetaData> PRFakeSource::getFormat() {
    return mOriginalMediaSource->getFormat();
}

status_t PRFakeSource::read(MediaBuffer **buffer, const ReadOptions *options) {
    Mutex::Autolock autoLock(mPRFakeLock);

    status_t err;
    int64_t seekTimeUs1;
    ReadOptions::SeekMode mode1;
    if (options && options->getSeekTo(&seekTimeUs1, &mode1) && mCodecSpecificData.size() != 0) {
        mOptions.setSeekTo(seekTimeUs1, mode1);
        mCodecSpecificDataIndex = 0;
        ALOGI("seek should send config data");
    }

    if (mCodecSpecificDataIndex < mCodecSpecificData.size()) {
        ALOGI("set codec info");
        const CodecSpecificData *specific =
            mCodecSpecificData[mCodecSpecificDataIndex];

        if (!mWantsNALFragments) {          // add nal prefix
            err = secureCopy((void *)specific, specific->mSize + 4, true);
            if (err != OK) {
                return err;
            }
        }  // else {    // Todo, default mWantsNALFragments false

        *buffer = mBuffer;
        mBuffer = NULL;
        mCodecSpecificDataIndex++;
        return OK;
    }

    ALOGV("zxy fakeMode %s(),line:%d", __FUNCTION__, __LINE__);
    int64_t tracetime_0 = systemTime()/1000;
    if (mOptions.getSeekTo(&seekTimeUs1, &mode1)) {
        err = mOriginalMediaSource->read(buffer, &mOptions);
        mOptions.clearSeekTo();
    } else {
        err = mOriginalMediaSource->read(buffer, options);
    }
    if (err != OK) {
        return err;
    }

    int64_t tracetime_1 = systemTime()/1000;
    if (mGroup != NULL) {           // only video would use mGroup
        err = secureCopy((void *)buffer, (*buffer)->range_length());
        if (err != OK) {
            return err;
        }
        (*buffer)->release();
        *buffer = mBuffer;
        mBuffer = NULL;
    }
    int64_t tracetime_2 = systemTime()/1000;
    ALOGV("extract read: %lld,secureCopy time %lld,total time: %lld",
            (long long)(tracetime_1-tracetime_0),(long long)(tracetime_2-tracetime_1),(long long)(tracetime_2-tracetime_0));

    return OK;
}

status_t PRFakeSource::secureCopy(void *src, size_t len, bool addPrefix) {
    if (mGroup == NULL) {
        ALOGE("mGroup is NULL");
        return -1;
    }
    int64_t tracetime_0 = systemTime()/1000;
    status_t err = mGroup->acquire_buffer(&mBuffer);
    int64_t tracetime_1 = systemTime()/1000;
    ALOGV("acquire_buffer: %lld", (long long)(tracetime_1-tracetime_0));
    if (err != OK) {
        CHECK(mBuffer == NULL);
        return err;
    }

    if (len > mBuffer->range_length()) {
        ALOGE("len:%zu is too large", len);
        return -1;
    }

    void *map_va = NULL;
    ssize_t index = mPaMap.indexOfKey((void *)(mBuffer->data()));
    if (index < 0) {
        ALOGE("Get map va failed, %zd", index);
        return UNKNOWN_ERROR;
    } else {
        map_va = mPaMap.valueAt(index).va;
    }

    uint8_t *src1 = (uint8_t *)(map_va);
    MediaBuffer **buffer = NULL;
    if (addPrefix) {
        CodecSpecificData *specific = (CodecSpecificData *)src;
        memcpy(src1, "\x00\x00\x00\x01", 4);
        memcpy(src1+4, specific->mData, specific->mSize);
        mBuffer->set_range(0, specific->mSize+4);

        const sp<MetaData> bufmeta = mBuffer->meta_data();
        bufmeta->clear();
        bufmeta->setInt64(kKeyTime, 0);
        return OK;
    } else {
        buffer = (MediaBuffer **)src;
        uint8_t *src0 = (uint8_t *)((*buffer)->data());
        ALOGV("len:%zu, before map dec:%02x %02x %02x %02x %02x", len, src0[0], src0[1], src0[2], src0[3], src0[4]);
        memcpy(src1, (*buffer)->data(), len);
        mBuffer->set_range(0, len);
        ALOGD("len:%zu, dec:%02x %02x %02x %02x %02x %02x %02x,bf:%02x %02x %02x %02x %02x", mBuffer->range_length(),
              src1[0], src1[1], src1[2], src1[3], src1[4], src1[5], src1[6], src0[0], src0[1], src0[2], src0[3], src0[4]);
    }

    // get meta info
    int64_t lastBufferTimeUs, targetSampleTimeUs;
    int32_t isSyncFrame;
    mBuffer->meta_data()->clear();
    CHECK((*buffer)->meta_data()->findInt64(kKeyTime, &lastBufferTimeUs));
    mBuffer->meta_data()->setInt64(kKeyTime, lastBufferTimeUs);

    if ((*buffer)->meta_data()->findInt64(kKeyTargetTime, &targetSampleTimeUs)) {
        mBuffer->meta_data()->setInt64(
                kKeyTargetTime, targetSampleTimeUs);
    }
    if ((*buffer)->meta_data()->findInt32(kKeyIsSyncFrame, &isSyncFrame)) {
        mBuffer->meta_data()->setInt32(kKeyIsSyncFrame, isSyncFrame);
    }
    return OK;
}

status_t PRFakeSource::setBuffers(const Vector<MediaBuffer *> &buffers) {
    mGroup = new MediaBufferGroup;
    for (size_t i = 0; i < buffers.size(); ++i) {
        ALOGI("mGroup add buffer:%zu, 0x%p", i, (buffers.itemAt(i))->data());
        mGroup->add_buffer(buffers.itemAt(i));
        MediaBuffer* buf = buffers.itemAt(i);

        ion_user_handle_t handle;
        int ion_fd = open("/dev/ion", O_RDONLY);
        if (ion_fd < 0) {
            ALOGE("open ion fail (%s)", strerror(errno));
            return UNKNOWN_ERROR;
        }
        const native_handle_t *native_handle_ptr =(native_handle_t *)(buf->data());
        int share_fd = native_handle_ptr->data[0];

        int ret = ion_import(ion_fd, share_fd, &handle);
        if (ret < 0) {
            ALOGE("ion import fail (%d)", ret);
            return UNKNOWN_ERROR;
        }

        // get secure handle, only for debug, no use
        // when use TPlayer, it would use it
#if 0
        {
            struct ion_mm_data mm_data;
            mm_data.mm_cmd = ION_MM_CONFIG_BUFFER;
            mm_data.config_buffer_param.handle = handle;
            mm_data.config_buffer_param.eModuleID = 1;
            mm_data.config_buffer_param.security = 0;
            mm_data.config_buffer_param.coherent = 1;
            if (ion_custom_ioctl(ion_fd, ION_CMD_MULTIMEDIA, &mm_data))
            {
                ALOGE("IOCTL[ION_IOC_CUSTOM] Config Buffer failed!\n");
                return UNKNOWN_ERROR;
            }
        }
        struct ion_sys_data sys_data;
        sys_data.sys_cmd = ION_SYS_GET_PHYS;
        sys_data.get_phys_param.handle = handle;
        if (ion_custom_ioctl(ion_fd, ION_CMD_SYSTEM, &sys_data)) {
            ALOGE("ion_custom_ioctl Get Phys failed!\n");
            return UNKNOWN_ERROR;
        }
        ALOGD("Physical address = 0x%x, len = %zd", sys_data.get_phys_param.phy_addr, sys_data.get_phys_param.len);
#endif
        // mmap get va
        size_t bufSize = buf->range_length();
        void *pBuf = ion_mmap(ion_fd, NULL, bufSize, PROT_READ|PROT_WRITE, MAP_SHARED, share_fd, 0);
        if (pBuf == NULL) {
            ALOGE("mmap fail (%s)", strerror(errno));
            return UNKNOWN_ERROR;
        }
        ALOGD("ion map ok buf: %p, share_fd:%d, handle:%d", pBuf, share_fd, handle);

        // save to vector
        VaMapStruct vaMap;
        vaMap.len = bufSize;
        vaMap.va = (void *)pBuf;
        vaMap.ion_fd = ion_fd;
        vaMap.share_fd = share_fd;
        vaMap.handle = handle;
        mPaMap.add((void *)native_handle_ptr, vaMap);
    }
    return OK;
}
////////////////////////////////////////////////////////////////////////////////

PRFakeExtractor::PRFakeExtractor(const sp<DataSource> &source, const char* mime)
    : mDataSource(source) {
    mOriginalExtractor = MediaExtractor::CreateFromService(source, mime);
    ALOGI("mime:%s", mime);
    if (mOriginalExtractor == NULL) {
        ALOGE("origi extractor is NULL");
    }
}

PRFakeExtractor::~PRFakeExtractor() {
}

size_t PRFakeExtractor::countTracks() {
    return mOriginalExtractor->countTracks();
}

sp<IMediaSource> PRFakeExtractor::getTrack(size_t index) {
    sp<IMediaSource> originalMediaSource = mOriginalExtractor->getTrack(index);
    const char *mime = NULL;
    CHECK(originalMediaSource->getFormat()->findCString(kKeyMIMEType, &mime));
    if (!strncasecmp("video/", mime, 6)) {
        ALOGI("playReady video track set kKeyRequiresSecureBuffers");
        originalMediaSource->getFormat()->setInt32(kKeyRequiresSecureBuffers, true);
    }
    return interface_cast<IMediaSource>(
            new PRFakeSource(originalMediaSource));
}

sp<MetaData> PRFakeExtractor::getTrackMetaData(size_t index, uint32_t flags) {
    return mOriginalExtractor->getTrackMetaData(index, flags);
}

sp<MetaData> PRFakeExtractor::getMetaData() {
    sp<MetaData> meta = mOriginalExtractor->getMetaData();
    // should set it, or else cause seek issue
    // Genericsource would use it for seek behavior
    meta->setInt32(kKeyIsPlayReady, true);
    return meta;
}

bool SniffPRFake(
    const sp<DataSource> &source, String8 *mimeType, float *confidence,
        sp<AMessage> *) {
    if (isFakeMode()) {
        uint8_t header[32];

        ssize_t n = source->readAt(0, header, sizeof(header));
        if (n < (ssize_t)sizeof(header)) {
            return false;
        }

        const uint8_t PlayReadyFakeID[12] = {
            '_', 'P', '_', 'R', '_', 'F', '_', 'M', '_', 'S', '_', 'V'};
        if (!memcmp((uint8_t *)header+4, "ftyp", 4) &&
            !memcmp((uint8_t *)header+8, PlayReadyFakeID, sizeof(PlayReadyFakeID))) {
            *mimeType = String8("prfakemode+") + MEDIA_MIMETYPE_CONTAINER_MPEG4;
            *confidence = 10.0f;
            ALOGI("Sniff PlayReady Fake File %s(),line:%d %s", __FUNCTION__, __LINE__, mimeType->string());
            return true;
        }
    }

    return false;
}
}  // namespace android
#endif
