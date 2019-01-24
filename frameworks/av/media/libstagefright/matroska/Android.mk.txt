LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_SRC_FILES:=                 \
        MatroskaExtractor.cpp

LOCAL_C_INCLUDES:= \
        $(TOP)/external/libvpx/libwebm \
        $(TOP)/frameworks/native/include/media/openmax \
        $(TOP)/frameworks/av/media/libstagefright/include \

LOCAL_CFLAGS += -Wno-multichar -Werror -Wall
LOCAL_CLANG := true
LOCAL_SANITIZE := unsigned-integer-overflow signed-integer-overflow

######################## MTK_USE_ANDROID_MM_DEFAULT_CODE ######################
ifeq ($(strip $(MTK_USE_ANDROID_MM_DEFAULT_CODE)),yes)
LOCAL_CFLAGS += -DANDROID_DEFAULT_CODE
else

LOCAL_C_INCLUDES += \
$(TOP)/$(MTK_ROOT)/frameworks/av/media/libstagefright/include/omx_core

endif  # MTK_USE_ANDROID_MM_DEFAULT_CODE
######################## MTK_USE_ANDROID_MM_DEFAULT_CODE ######################


ifeq ($(MTK_MKV_PLAYBACK_ENHANCEMENT),yes)
LOCAL_CFLAGS += -DMTK_MKV_PLAYBACK_ENHANCEMENT
endif

LOCAL_MODULE:= libstagefright_matroska

include $(MTK_STATIC_LIBRARY)
