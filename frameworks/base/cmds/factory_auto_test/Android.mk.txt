LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_SRC_FILES:= factory_auto_test

LOCAL_MODULE:= factory_auto_test
LOCAL_MODULE_TAGS := optional
LOCAL_MODULE_CLASS:= EXECUTABLES
LOCAL_MODULE_PATH := $(TARGET_OUT)/bin

include $(BUILD_PREBUILT)
