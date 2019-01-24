LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)

LOCAL_CLANG := true

LOCAL_ADDITIONAL_DEPENDENCIES := $(LOCAL_PATH)/Android.mk
LOCAL_SRC_FILES := \
    Client.cpp \
    DisplayDevice.cpp \
    DispSync.cpp \
    EventControlThread.cpp \
    EventThread.cpp \
    FenceTracker.cpp \
    FrameTracker.cpp \
    GpuService.cpp \
    Layer.cpp \
    LayerDim.cpp \
    MessageQueue.cpp \
    MonitoredProducer.cpp \
    SurfaceFlingerConsumer.cpp \
    Transform.cpp \
    DisplayHardware/FramebufferSurface.cpp \
    DisplayHardware/HWC2.cpp \
    DisplayHardware/HWC2On1Adapter.cpp \
    DisplayHardware/PowerHAL.cpp \
    DisplayHardware/VirtualDisplaySurface.cpp \
    Effects/Daltonizer.cpp \
    EventLog/EventLogTags.logtags \
    EventLog/EventLog.cpp \
    RenderEngine/Description.cpp \
    RenderEngine/Mesh.cpp \
    RenderEngine/Program.cpp \
    RenderEngine/ProgramCache.cpp \
    RenderEngine/GLExtensions.cpp \
    RenderEngine/RenderEngine.cpp \
    RenderEngine/Texture.cpp \
    RenderEngine/GLES10RenderEngine.cpp \
    RenderEngine/GLES11RenderEngine.cpp \
    RenderEngine/GLES20RenderEngine.cpp

LOCAL_C_INCLUDES := \
	frameworks/native/vulkan/include \
	external/vulkan-validation-layers/libs/vkjson

LOCAL_CFLAGS := -DLOG_TAG=\"SurfaceFlinger\"
LOCAL_CFLAGS += -DGL_GLEXT_PROTOTYPES -DEGL_EGLEXT_PROTOTYPES
#LOCAL_CFLAGS += -DENABLE_FENCE_TRACKING

USE_HWC2 := false
ifeq ($(USE_HWC2),true)
    LOCAL_CFLAGS += -DUSE_HWC2
    LOCAL_SRC_FILES += \
        SurfaceFlinger.cpp \
        DisplayHardware/HWComposer.cpp
else
    LOCAL_SRC_FILES += \
        SurfaceFlinger_hwc1.cpp \
        DisplayHardware/HWComposer_hwc1.cpp
endif

ifeq ($(TARGET_BOARD_PLATFORM),omap4)
    LOCAL_CFLAGS += -DHAS_CONTEXT_PRIORITY
endif
ifeq ($(TARGET_BOARD_PLATFORM),s5pc110)
    LOCAL_CFLAGS += -DHAS_CONTEXT_PRIORITY
endif

ifeq ($(TARGET_DISABLE_TRIPLE_BUFFERING),true)
    LOCAL_CFLAGS += -DTARGET_DISABLE_TRIPLE_BUFFERING
endif

ifeq ($(TARGET_FORCE_HWC_FOR_VIRTUAL_DISPLAYS),true)
    LOCAL_CFLAGS += -DFORCE_HWC_COPY_FOR_VIRTUAL_DISPLAYS
endif

ifneq ($(NUM_FRAMEBUFFER_SURFACE_BUFFERS),)
    LOCAL_CFLAGS += -DNUM_FRAMEBUFFER_SURFACE_BUFFERS=$(NUM_FRAMEBUFFER_SURFACE_BUFFERS)
endif

ifeq ($(TARGET_RUNNING_WITHOUT_SYNC_FRAMEWORK),true)
    LOCAL_CFLAGS += -DRUNNING_WITHOUT_SYNC_FRAMEWORK
endif

# See build/target/board/generic/BoardConfig.mk for a description of this setting.
ifneq ($(VSYNC_EVENT_PHASE_OFFSET_NS),)
    LOCAL_CFLAGS += -DVSYNC_EVENT_PHASE_OFFSET_NS=$(VSYNC_EVENT_PHASE_OFFSET_NS)
else
    LOCAL_CFLAGS += -DVSYNC_EVENT_PHASE_OFFSET_NS=0
endif

# See build/target/board/generic/BoardConfig.mk for a description of this setting.
ifneq ($(SF_VSYNC_EVENT_PHASE_OFFSET_NS),)
    LOCAL_CFLAGS += -DSF_VSYNC_EVENT_PHASE_OFFSET_NS=$(SF_VSYNC_EVENT_PHASE_OFFSET_NS)
else
    LOCAL_CFLAGS += -DSF_VSYNC_EVENT_PHASE_OFFSET_NS=0
endif

ifneq ($(PRESENT_TIME_OFFSET_FROM_VSYNC_NS),)
    LOCAL_CFLAGS += -DPRESENT_TIME_OFFSET_FROM_VSYNC_NS=$(PRESENT_TIME_OFFSET_FROM_VSYNC_NS)
else
    LOCAL_CFLAGS += -DPRESENT_TIME_OFFSET_FROM_VSYNC_NS=0
endif

ifneq ($(MAX_VIRTUAL_DISPLAY_DIMENSION),)
    LOCAL_CFLAGS += -DMAX_VIRTUAL_DISPLAY_DIMENSION=$(MAX_VIRTUAL_DISPLAY_DIMENSION)
else
    LOCAL_CFLAGS += -DMAX_VIRTUAL_DISPLAY_DIMENSION=0
endif

LOCAL_CFLAGS += -fvisibility=hidden -Werror=format
LOCAL_CFLAGS += -std=c++14

LOCAL_STATIC_LIBRARIES := libvkjson
LOCAL_SHARED_LIBRARIES := \
    libcutils \
    liblog \
    libdl \
    libhardware \
    libutils \
    libEGL \
    libGLESv1_CM \
    libGLESv2 \
    libbinder \
    libui \
    libgui \
    libpowermanager \
    libvulkan

# --- MediaTek ---------------------------------------------------------------
ifneq (, $(findstring MTK_AOSP_ENHANCEMENT, $(MTK_GLOBAL_CFLAGS)))
	LOCAL_SRC_FILES += \
		mediatek/DisplayDevice.cpp \
		mediatek/SurfaceFlinger.cpp \
		mediatek/RenderEngine/RenderEngine.cpp \
		mediatek/RenderEngine/GLES11RenderEngine.cpp \
		mediatek/RenderEngine/GLES20RenderEngine.cpp \
		mediatek/SurfaceFlingerWatchDog.cpp \
		mediatek/Layer.cpp \
		mediatek/MtkHwc.cpp \
		mediatek/Resync.cpp

ifeq ($(MTK_GLOBAL_PQ_SUPPORT),yes)
    LOCAL_SRC_FILES += mediatek/SurfaceFlingerPQ.cpp
    LOCAL_CFLAGS += -DMTK_GLOBAL_PQ_SUPPORT
    LOCAL_C_INCLUDES += \
        $(TOP)/$(MTK_ROOT)/hardware/pq/v2.0/include
    LOCAL_SHARED_LIBRARIES += libpqservice
endif

ifneq ($(strip $(TARGET_BUILD_VARIANT)), eng)
	LOCAL_CFLAGS += -DMTK_USER_BUILD
endif

ifeq ($(MTK_EMULATOR_SUPPORT), yes)
	LOCAL_CFLAGS += -DMTK_EMULATOR_SUPPORT
endif

ifeq ($(FPGA_EARLY_PORTING), yes)
	LOCAL_CFLAGS += -DFPGA_EARLY_PORTING
endif

ifeq ($(HAVE_AEE_FEATURE), yes)
	LOCAL_CFLAGS += -DHAVE_AEE_FEATURE
	LOCAL_SHARED_LIBRARIES += libaed
endif

	LOCAL_REQUIRED_MODULES += \
		drm_disable_icon.png

	LOCAL_SHARED_LIBRARIES += \
		libskia \
		libui_ext \
		libselinux \
		libgralloc_extra

	LOCAL_C_INCLUDES += \
		$(TOP)/$(MTK_ROOT)/hardware/hwcomposer/include \
		$(TOP)/$(MTK_ROOT)/hardware/libgem/inc \
		$(TOP)/$(MTK_ROOT)/hardware/gralloc_extra/include \
		$(TOP)/$(MTK_ROOT)/external/aee/binary/inc

	# these values shall sync with HWC
	LOCAL_CFLAGS += -DMTK_HWC_THRESHOLD_VIDEO=3840*2160
	LOCAL_CFLAGS += -DMTK_HWC_THRESHOLD_DISPLAY=3840*2160

ifneq ($(MTK_EMULATOR_SUPPORT), yes)
	LOCAL_SHARED_LIBRARIES += \
		libged
	LOCAL_C_INCLUDES += \
		$(TOP)/$(MTK_ROOT)/hardware/gpu/include
endif
endif
# ----------------------------------------------------------------------------


LOCAL_MODULE := libsurfaceflinger

LOCAL_CFLAGS += -Wall -Werror -Wunused -Wunreachable-code

include $(MTK_SHARED_LIBRARY)

###############################################################
# build surfaceflinger's executable
include $(CLEAR_VARS)

LOCAL_CLANG := true

LOCAL_LDFLAGS := -Wl,--version-script,art/sigchainlib/version-script.txt -Wl,--export-dynamic
LOCAL_CFLAGS := -DLOG_TAG=\"SurfaceFlinger\"
LOCAL_CPPFLAGS := -std=c++14

LOCAL_INIT_RC := surfaceflinger.rc

ifneq ($(ENABLE_CPUSETS),)
    LOCAL_CFLAGS += -DENABLE_CPUSETS
endif

LOCAL_SRC_FILES := \
    main_surfaceflinger.cpp

LOCAL_SHARED_LIBRARIES := \
    libsurfaceflinger \
    libcutils \
    liblog \
    libbinder \
    libutils \
    libdl

LOCAL_WHOLE_STATIC_LIBRARIES := libsigchain

# --- MediaTek ---------------------------------------------------------------
ifneq (, $(findstring MTK_AOSP_ENHANCEMENT, $(MTK_GLOBAL_CFLAGS)))
	LOCAL_C_INCLUDES := \
		$(TOP)/$(MTK_ROOT)/hardware/hwcomposer/include \
		$(TOP)/$(MTK_ROOT)/hardware/include \
		$(TOP)/$(MTK_ROOT)/hardware/libgem/inc

	LOCAL_SHARED_LIBRARIES += \
		libgui_ext

	LOCAL_CFLAGS += -DMTK_AOSP_ENHANCEMENT
endif
# ----------------------------------------------------------------------------

LOCAL_MODULE := surfaceflinger

ifdef TARGET_32_BIT_SURFACEFLINGER
LOCAL_32_BIT_ONLY := true
endif

LOCAL_CFLAGS += -Wall -Werror -Wunused -Wunreachable-code

include $(BUILD_EXECUTABLE)

###############################################################
# uses jni which may not be available in PDK
ifneq ($(wildcard libnativehelper/include),)
include $(CLEAR_VARS)

LOCAL_CLANG := true

LOCAL_CFLAGS := -DLOG_TAG=\"SurfaceFlinger\"
LOCAL_CPPFLAGS := -std=c++14

LOCAL_SRC_FILES := \
    DdmConnection.cpp

LOCAL_SHARED_LIBRARIES := \
    libcutils \
    liblog \
    libdl

LOCAL_MODULE := libsurfaceflinger_ddmconnection

LOCAL_CFLAGS += -Wall -Werror -Wunused -Wunreachable-code

include $(BUILD_SHARED_LIBRARY)
endif # libnativehelper
