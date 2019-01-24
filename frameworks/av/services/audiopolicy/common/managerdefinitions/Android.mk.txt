LOCAL_PATH:= $(call my-dir)

include $(CLEAR_VARS)

LOCAL_SRC_FILES:= \
    src/DeviceDescriptor.cpp \
    src/AudioGain.cpp \
    src/HwModule.cpp \
    src/IOProfile.cpp \
    src/AudioPort.cpp \
    src/AudioProfile.cpp \
    src/AudioRoute.cpp \
    src/AudioPolicyMix.cpp \
    src/AudioPatch.cpp \
    src/AudioInputDescriptor.cpp \
    src/AudioOutputDescriptor.cpp \
    src/AudioCollections.cpp \
    src/EffectDescriptor.cpp \
    src/SoundTriggerSession.cpp \
    src/SessionRoute.cpp \
    src/AudioSourceDescriptor.cpp \
    src/VolumeCurve.cpp \
    src/TypeConverter.cpp \
    src/AudioSession.cpp

LOCAL_SHARED_LIBRARIES := \
    libcutils \
    libutils \
    liblog \

LOCAL_C_INCLUDES := \
    $(LOCAL_PATH)/include \
    $(TOPDIR)frameworks/av/services/audiopolicy/common/include \
    $(TOPDIR)frameworks/av/services/audiopolicy \
    $(TOPDIR)frameworks/av/services/audiopolicy/utilities \

ifeq ($(USE_XML_AUDIO_POLICY_CONF), 1)

LOCAL_SRC_FILES += src/Serializer.cpp

LOCAL_STATIC_LIBRARIES += libxml2

LOCAL_SHARED_LIBRARIES += libicuuc

LOCAL_C_INCLUDES += \
    $(TOPDIR)external/libxml2/include \
    $(TOPDIR)external/icu/icu4c/source/common

else

LOCAL_SRC_FILES += \
    src/ConfigParsingUtils.cpp \
    src/StreamDescriptor.cpp \
    src/Gains.cpp

endif #ifeq ($(USE_XML_AUDIO_POLICY_CONF), 1)

ifeq ($(MTK_AUDIO),yes)
LOCAL_CFLAGS += -DMTK_AUDIO

ifeq ($(strip $(MTK_CROSSMOUNT_SUPPORT)),yes)
LOCAL_C_INCLUDES += \
    $(MTK_PATH_SOURCE)/protect-bsp/frameworks/av/media/libstagefright/cross-mount \
    $(TOP)/frameworks/av/include/camera \
    $(MTK_PATH_SOURCE)/hardware/include/ \
    $(TOP)/system/media/camera/include/
LOCAL_SHARED_LIBRARIES += libcrossmount
endif

ifeq ($(strip $(TARGET_BUILD_VARIANT)),eng)
  LOCAL_CFLAGS += -DCONFIG_MT_ENG_BUILD
endif

ifeq ($(strip $(MTK_HIGH_RESOLUTION_AUDIO_SUPPORT)),yes)
    LOCAL_CFLAGS += -DMTK_HD_AUDIO_ARCHITECTURE
endif

ifeq ($(DISABLE_EARPIECE),yes)
    LOCAL_CFLAGS += -DDISABLE_EARPIECE
endif

ifeq ($(MTK_SUPPORT_TC1_TUNNING),yes)
  LOCAL_CFLAGS += -DMTK_AUDIO_GAIN_TABLE
endif

ifeq ($(strip $(MTK_FM_SUPPORT)),yes)
    ifeq ($(strip $(MTK_FM_TX_SUPPORT)),yes)
        ifeq ($(strip $(MTK_FM_TX_AUDIO)),FM_DIGITAL_OUTPUT)
            LOCAL_CFLAGS += -DFM_DIGITAL_OUT_SUPPORT
        endif
    endif
endif


ifeq ($(HAVE_AEE_FEATURE),yes)
    LOCAL_SHARED_LIBRARIES += libaed
    LOCAL_C_INCLUDES += \
    $(MTK_PATH_SOURCE)/external/aee/binary/inc
    LOCAL_CFLAGS += -DHAVE_AEE_FEATURE
endif

ifeq ($(MTK_DOLBY_DAP_SUPPORT), yes)
        LOCAL_CFLAGS += -DDOLBY_DAP_OPENSLES
        LOCAL_CFLAGS += -DDOLBY_DAP_OPENSLES_MOVE_EFFECT
        LOCAL_C_INCLUDES += $(TOP)/vendor/dolby/ds1/libds/include/
endif

LOCAL_SHARED_LIBRARIES += \
    libmedia \
    libaudiocustparam

LOCAL_SRC_FILES += \
    src/AudioHeadsetDetect.cpp
endif

LOCAL_C_INCLUDES += \
    $(TOPDIR)/frameworks/av/include \
    $(MTK_PATH_PLATFORM)/hardware/audio/include \
    $(MTK_PATH_SOURCE)/hardware/audio/common/include \
    $(MTK_PATH_SOURCE)/external/nvram/libnvram \
    $(MTK_PATH_SOURCE)/external/AudioCompensationFilter \
    $(MTK_PATH_SOURCE)/external/AudioComponentEngine \
    $(MTK_PATH_SOURCE)/external/HeadphoneCompensationFilter \
    $(MTK_PATH_SOURCE)/external/audiocustparam \
    $(MTK_PATH_SOURCE)/frameworks/av/include/media \
    $(MTK_PATH_SOURCE)/frameworks/av/include \
    $(TOP)/frameworks/av/include/media \
    $(MTK_PATH_CUSTOM)/custom \
    $(MTK_PATH_CUSTOM)/custom/audio \
    $(MTK_PATH_CUSTOM)/cgen/inc \
    $(MTK_PATH_CUSTOM)/cgen/cfgfileinc \
    $(MTK_PATH_CUSTOM)/cgen/cfgdefault \
    $(MTK_PATH_CUSTOM)/../common/cgen/inc \
    $(MTK_PATH_CUSTOM)/../common/cgen/cfgfileinc \
    $(MTK_PATH_CUSTOM)/../common/cgen/cfgdefault \
    $(MTK_PATH_CUSTOM)/hal/audioflinger/audio

LOCAL_EXPORT_C_INCLUDE_DIRS := \
    $(LOCAL_PATH)/include

LOCAL_MULTILIB := $(AUDIOSERVER_MULTILIB)

LOCAL_MODULE := libaudiopolicycomponents

include $(BUILD_STATIC_LIBRARY)
