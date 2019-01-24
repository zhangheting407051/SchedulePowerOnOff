LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

ifneq ($(strip $(MTK_USE_ANDROID_MM_DEFAULT_CODE)),yes)
ifeq ($(strip $(MTK_DP_FRAMEWORK)),yes)
LOCAL_CFLAGS += -DMTK_USEDPFRMWK
else
LOCAL_CFLAGS += -DMTK_MHAL
endif
endif

LOCAL_SRC_FILES:=                     \
        ColorConverter.cpp            \
        SoftwareRenderer.cpp

LOCAL_CFLAGS += -Werror
LOCAL_CLANG := true
LOCAL_SANITIZE := signed-integer-overflow

LOCAL_C_INCLUDES := \
    $(TOP)/$(MTK_ROOT)/frameworks/native/include/media/openmax \
    $(TOP)/hardware/msm7k \
	$(TOP)/vendor/mediatek/proprietary/hardware/dpframework/inc \
	$(TOP)/system/core/include/system \
	$(TOP)/vendor/mediatek/proprietary/external/mhal/inc
ifneq ($(strip $(MTK_EMULATOR_SUPPORT)),yes)
LOCAL_SHARED_LIBRARIES := \
    libdpframework
endif
    
LOCAL_MODULE:= libstagefright_color_conversion

include $(MTK_STATIC_LIBRARY)
