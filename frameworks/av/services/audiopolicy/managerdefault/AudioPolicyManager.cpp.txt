/*
 * Copyright (C) 2009 The Android Open Source Project
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

#define LOG_TAG "APM_AudioPolicyManager"
//#define LOG_NDEBUG 0
#ifdef MTK_AUDIO
#define LOG_NDEBUG 0
#include <cutils/log.h>
#ifdef CONFIG_MT_ENG_BUILD
#define VERY_VERBOSE_LOGGING
#endif
#else
#include <utils/Log.h>
#endif

//#define VERY_VERBOSE_LOGGING
#ifdef VERY_VERBOSE_LOGGING
#define ALOGVV ALOGV
#else
#define ALOGVV(a...) do { } while(0)
#endif

#define AUDIO_POLICY_XML_CONFIG_FILE "/system/etc/audio_policy_configuration.xml"

#include <inttypes.h>
#include <math.h>

#include <AudioPolicyManagerInterface.h>
#include <AudioPolicyEngineInstance.h>
#include <cutils/properties.h>
#include <utils/Log.h>
#ifdef MTK_AUDIO
#include <hardware/audio_mtk.h>
#include <hardware/audio_effect_mtk.h>
#else
#include <hardware/audio.h>
#include <hardware/audio_effect.h>
#endif
#include <media/AudioParameter.h>
#include <media/AudioPolicyHelper.h>
#include <soundtrigger/SoundTrigger.h>
#include "AudioPolicyManager.h"
#ifndef USE_XML_AUDIO_POLICY_CONF
#include <ConfigParsingUtils.h>
#include <StreamDescriptor.h>
#endif
#include <Serializer.h>
#include "TypeConverter.h"
#include <policy.h>
#if defined(DOLBY_UDC) || defined(DOLBY_DAP_MOVE_EFFECT)
#include "DolbyAudioPolicy_impl.h"
#endif // DOLBY_END
#ifdef MTK_AUDIO
#include <media/mediarecorder.h>
#ifdef MTK_CROSSMOUNT_SUPPORT
#include <sink/CrossMountSink.h>
#endif
#include "AudioDef.h"
#include "audio_custom_exp.h"
#include "AudioPolicyParameters.h"
#define MUSIC_WAIT_TIME (1000*200)
#ifndef BOOT_ANIMATION_VOLUME
#define  BOOT_ANIMATION_VOLUME (0.25)
#endif
#ifdef MTK_NEW_VOL_CONTROL
#include "AudioParamParser.h"
#include <media/IAudioPolicyService.h>
namespace android {
void gainTableXmlChangedCb(AppHandle *_appHandle, const char *_audioTypeName);
};
#endif
static   const char * gaf_policy_r_submix_propty = "af.policy.r_submix_prio_adjust";
static   const char PROPERTY_KEY_FM_FORCE_DIRECT_MODE_TYPE[PROPERTY_KEY_MAX]  = "af.fm.force_direct_mode_type";
static   const char PROPERTY_KEY_POLICY_MODE[PROPERTY_KEY_MAX]  = "af.Policy.SampleRate_policy";
static   const char PROPERTY_KEY_POLICY_DEBUG[PROPERTY_KEY_MAX]  = "af.policy.debug";
#ifndef VOICE_VOLUME_MAX
#define VOICE_VOLUME_MAX       (160)
#endif
#ifndef VOICE_ONEDB_STEP
#define VOICE_ONEDB_STEP         (4)
#endif
#define SUPPORT_ANDROID_FM_PLAYER   //Using Android FM Player
#endif
// shouldn't need to touch these
//static const float dBConvert = -dBPerStep * 2.302585093f / 20.0f;
//static const float dBConvertInverse = 1.0f / dBConvert;
#ifdef MTK_AUDIO_GAIN_TABLE
// total 63.5 dB
static const float KeydBPerStep = 0.25f;
static const float KeyvolumeStep = 255.0f;
// shouldn't need to touch these
static const float KeydBConvert = -KeydBPerStep * 2.302585093f / 20.0f;
static const float KeydBConvertInverse = 1.0f / KeydBPerStep;
#endif
#ifdef MTK_AUDIO
static int gMTKPolicyLoglevel = 0;
#define MTK_ALOGVV(...) if (gMTKPolicyLoglevel > 1) __android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, __VA_ARGS__)
#define MTK_ALOGV(...) if (gMTKPolicyLoglevel > 0) __android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, __VA_ARGS__)
#else
#define MTK_ALOGVV(...) do { } while(0)
#define MTK_ALOGV(...) do { } while(0)
#endif

namespace android {

// ----------------------------------------------------------------------------
// AudioPolicyInterface implementation
// ----------------------------------------------------------------------------

status_t AudioPolicyManager::setDeviceConnectionState(audio_devices_t device,
                                                      audio_policy_dev_state_t state,
                                                      const char *device_address,
                                                      const char *device_name)
{
    return setDeviceConnectionStateInt(device, state, device_address, device_name);
}

status_t AudioPolicyManager::setDeviceConnectionStateInt(audio_devices_t device,
                                                         audio_policy_dev_state_t state,
                                                         const char *device_address,
                                                         const char *device_name)
{
    ALOGV("setDeviceConnectionStateInt() device: 0x%X, state %d, address %s name %s",
            device, state, device_address, device_name);

    // connect/disconnect only 1 device at a time
    if (!audio_is_output_device(device) && !audio_is_input_device(device)) return BAD_VALUE;

    sp<DeviceDescriptor> devDesc =
            mHwModules.getDeviceDescriptor(device, device_address, device_name);

    // handle output devices
    if (audio_is_output_device(device)) {
        SortedVector <audio_io_handle_t> outputs;

        ssize_t index = mAvailableOutputDevices.indexOf(devDesc);

        // save a copy of the opened output descriptors before any output is opened or closed
        // by checkOutputsForDevice(). This will be needed by checkOutputForAllStrategies()
        mPreviousOutputs = mOutputs;
        switch (state)
        {
        // handle output device connection
        case AUDIO_POLICY_DEVICE_STATE_AVAILABLE: {
            if (index >= 0) {
#ifdef MTK_AUDIO    // The system notify APS later if headset is inserted before boot. We should make sure return true
                if (mNativeAutoDetectHeadSetFlag == true) {
                    ALOGW("setDeviceConnectionState() device already connected: %x, but return OK", device);
                    mNativeAutoDetectHeadSetFlag = false;
                    return NO_ERROR;
                }
#endif
                ALOGW("setDeviceConnectionState() device already connected: %x", device);
                return INVALID_OPERATION;
            }
            ALOGV("setDeviceConnectionState() connecting device %x", device);

            // register new device as available
            index = mAvailableOutputDevices.add(devDesc);
            if (index >= 0) {
                sp<HwModule> module = mHwModules.getModuleForDevice(device);
                if (module == 0) {
                    ALOGD("setDeviceConnectionState() could not find HW module for device %08x",
                          device);
                    mAvailableOutputDevices.remove(devDesc);
                    return INVALID_OPERATION;
                }
                mAvailableOutputDevices[index]->attach(module);
            } else {
                return NO_MEMORY;
            }

            if (checkOutputsForDevice(devDesc, state, outputs, devDesc->mAddress) != NO_ERROR) {
                mAvailableOutputDevices.remove(devDesc);
                return INVALID_OPERATION;
            }
            // Propagate device availability to Engine
            mEngine->setDeviceConnectionState(devDesc, state);

            // outputs should never be empty here
            ALOG_ASSERT(outputs.size() != 0, "setDeviceConnectionState():"
                    "checkOutputsForDevice() returned no outputs but status OK");
            ALOGV("setDeviceConnectionState() checkOutputsForDevice() returned %zu outputs",
                  outputs.size());

            // Send connect to HALs
            AudioParameter param = AudioParameter(devDesc->mAddress);
            param.addInt(String8(AUDIO_PARAMETER_DEVICE_CONNECT), device);
            mpClientInterface->setParameters(AUDIO_IO_HANDLE_NONE, param.toString());

            } break;
        // handle output device disconnection
        case AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE: {
            if (index < 0) {
                ALOGW("setDeviceConnectionState() device not connected: %x", device);
                return INVALID_OPERATION;
            }

            ALOGV("setDeviceConnectionState() disconnecting output device %x", device);

            // Send Disconnect to HALs
            AudioParameter param = AudioParameter(devDesc->mAddress);
            param.addInt(String8(AUDIO_PARAMETER_DEVICE_DISCONNECT), device);
            mpClientInterface->setParameters(AUDIO_IO_HANDLE_NONE, param.toString());

            // remove device from available output devices
            mAvailableOutputDevices.remove(devDesc);

            checkOutputsForDevice(devDesc, state, outputs, devDesc->mAddress);

            // Propagate device availability to Engine
            mEngine->setDeviceConnectionState(devDesc, state);
            } break;

        default:
            ALOGE("setDeviceConnectionState() invalid state: %x", state);
            return BAD_VALUE;
        }

        // checkA2dpSuspend must run before checkOutputForAllStrategies so that A2DP
        // output is suspended before any tracks are moved to it
        checkA2dpSuspend();
        checkOutputForAllStrategies();
        // outputs must be closed after checkOutputForAllStrategies() is executed
        if (!outputs.isEmpty()) {
            for (size_t i = 0; i < outputs.size(); i++) {
                sp<SwAudioOutputDescriptor> desc = mOutputs.valueFor(outputs[i]);
                // close unused outputs after device disconnection or direct outputs that have been
                // opened by checkOutputsForDevice() to query dynamic parameters
                if ((state == AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE) ||
                        (((desc->mFlags & AUDIO_OUTPUT_FLAG_DIRECT) != 0) &&
                         (desc->mDirectOpenCount == 0))) {
                    closeOutput(outputs[i]);
                }
            }
            // check again after closing A2DP output to reset mA2dpSuspended if needed
            checkA2dpSuspend();
        }

        updateDevicesAndOutputs();
#ifdef DOLBY_UDC
        // Before closing the opened outputs, update endpoint property with device capabilities
        audio_devices_t audioOutputDevice = getDeviceForStrategy(getStrategy(AUDIO_STREAM_MUSIC), true);
        mDolbyAudioPolicy.setEndpointSystemProperty(audioOutputDevice, mHwModules);
#endif // DOLBY_END
#ifdef MTK_AUDIO
        if (mAudioPolicyVendorControl.isStateInCallOnly(mEngine->getPhoneState()) && hasPrimaryOutput())
#else
        if (mEngine->getPhoneState() == AUDIO_MODE_IN_CALL && hasPrimaryOutput())
#endif
        {

            audio_devices_t newDevice = getNewOutputDevice(mPrimaryOutput, false /*fromCache*/);
            updateCallRouting(newDevice);
        }
        for (size_t i = 0; i < mOutputs.size(); i++) {
            sp<SwAudioOutputDescriptor> desc = mOutputs.valueAt(i);
#ifdef MTK_AUDIO
            if (!mAudioPolicyVendorControl.isStateInCallOnly(mEngine->getPhoneState()) || (desc != mPrimaryOutput))
#else
            if ((mEngine->getPhoneState() != AUDIO_MODE_IN_CALL) || (desc != mPrimaryOutput))
#endif
            {

                audio_devices_t newDevice = getNewOutputDevice(desc, true /*fromCache*/);
                // do not force device change on duplicated output because if device is 0, it will
                // also force a device 0 for the two outputs it is duplicated to which may override
                // a valid device selection on those outputs.
                bool force = !desc->isDuplicated()
                        && (!device_distinguishes_on_address(device)
                                // always force when disconnecting (a non-duplicated device)
                                || (state == AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE));
                setOutputDevice(desc, newDevice, force, 0);
            }
        }

        if (state == AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE) {
            cleanUpForDevice(devDesc);
        }

        mpClientInterface->onAudioPortListUpdate();
#ifdef MTK_AUDIO  // headset is inserted before boot, for boot animation. We need notify APS headset inserted ASAP.
        if (mNativeAutoDetectHeadSetFlag == false) {
            if (strcmp(device_name, "AudioNativeAutoDetect") == 0) {
                ALOGD("Get AudioNativeAutoDetect");
                mNativeAutoDetectHeadSetFlag = true;
            }
        }
        ALOGD("%s#Line%d NO_ERROR",__FUNCTION__,__LINE__);
#endif
        return NO_ERROR;
    }  // end if is output device

    // handle input devices
    if (audio_is_input_device(device)) {
        SortedVector <audio_io_handle_t> inputs;

        ssize_t index = mAvailableInputDevices.indexOf(devDesc);
        switch (state)
        {
        // handle input device connection
        case AUDIO_POLICY_DEVICE_STATE_AVAILABLE: {
            if (index >= 0) {
                ALOGW("setDeviceConnectionState() device already connected: %d", device);
                return INVALID_OPERATION;
            }
            sp<HwModule> module = mHwModules.getModuleForDevice(device);
            if (module == NULL) {
                ALOGW("setDeviceConnectionState(): could not find HW module for device %08x",
                      device);
                return INVALID_OPERATION;
            }
            if (checkInputsForDevice(devDesc, state, inputs, devDesc->mAddress) != NO_ERROR) {
                return INVALID_OPERATION;
            }

            index = mAvailableInputDevices.add(devDesc);
            if (index >= 0) {
                mAvailableInputDevices[index]->attach(module);
            } else {
                return NO_MEMORY;
            }

            // Set connect to HALs
            AudioParameter param = AudioParameter(devDesc->mAddress);
            param.addInt(String8(AUDIO_PARAMETER_DEVICE_CONNECT), device);
            mpClientInterface->setParameters(AUDIO_IO_HANDLE_NONE, param.toString());

            // Propagate device availability to Engine
            mEngine->setDeviceConnectionState(devDesc, state);
        } break;

        // handle input device disconnection
        case AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE: {
            if (index < 0) {
                ALOGW("setDeviceConnectionState() device not connected: %d", device);
                return INVALID_OPERATION;
            }

            ALOGV("setDeviceConnectionState() disconnecting input device %x", device);

            // Set Disconnect to HALs
            AudioParameter param = AudioParameter(devDesc->mAddress);
            param.addInt(String8(AUDIO_PARAMETER_DEVICE_DISCONNECT), device);
            mpClientInterface->setParameters(AUDIO_IO_HANDLE_NONE, param.toString());

            checkInputsForDevice(devDesc, state, inputs, devDesc->mAddress);
            mAvailableInputDevices.remove(devDesc);

            // Propagate device availability to Engine
            mEngine->setDeviceConnectionState(devDesc, state);
        } break;

        default:
            ALOGE("setDeviceConnectionState() invalid state: %x", state);
            return BAD_VALUE;
        }

#ifndef MTK_CROSSMOUNT_SUPPORT
        closeAllInputs();
#else
        if (strcmp(device_address, "CrossMount-Mic") == 0 &&
            !mAudioPolicyVendorControl.getCrossMountMicAudioMixerEnable()) {
            // When unregisterPolicyMixes "CrossMount-Mic", do not closeAllInputs().
            // This may cause RecordTrack switch to non-active.
        }else{
            closeAllInputs();
        }
#endif


#ifdef MTK_AUDIO    // Notify A2DP HAL that there is a input device is connected. A2DP will design A2DP-Sync
         if (audio_is_input_device(device) && audio_is_a2dp_in_device(device)) {
            if (state == AUDIO_POLICY_DEVICE_STATE_AVAILABLE) {
                ALOGD("send a2dp_in_dev_connect=1");
                String8 paramStr;
                paramStr = String8("a2dp_in_dev_connect=1");
                mpClientInterface->setParameters(AUDIO_IO_HANDLE_NONE, paramStr);
            }
            else if (state == AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE) {
                ALOGD("send a2dp_in_dev_connect=0");
                String8 paramStr;
                paramStr = String8("a2dp_in_dev_connect=0");
                mpClientInterface->setParameters(AUDIO_IO_HANDLE_NONE, paramStr);
            }
        }
        if (mAudioPolicyVendorControl.isStateInCallOnly(mEngine->getPhoneState()) && hasPrimaryOutput())
#else
        if (mEngine->getPhoneState() == AUDIO_MODE_IN_CALL && hasPrimaryOutput())
#endif
        {

            audio_devices_t newDevice = getNewOutputDevice(mPrimaryOutput, false /*fromCache*/);
            updateCallRouting(newDevice);
        }

        if (state == AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE) {
            cleanUpForDevice(devDesc);
        }

        mpClientInterface->onAudioPortListUpdate();
#ifdef MTK_AUDIO
        ALOGD("%s#Line%d NO_ERROR",__FUNCTION__,__LINE__);
#endif
        return NO_ERROR;
    } // end if is input device

    ALOGW("setDeviceConnectionState() invalid device: %x", device);
    return BAD_VALUE;
}

audio_policy_dev_state_t AudioPolicyManager::getDeviceConnectionState(audio_devices_t device,
                                                                      const char *device_address)
{
    sp<DeviceDescriptor> devDesc =
            mHwModules.getDeviceDescriptor(device, device_address, "",
                                           (strlen(device_address) != 0)/*matchAddress*/);

    if (devDesc == 0) {
        ALOGW("getDeviceConnectionState() undeclared device, type %08x, address: %s",
              device, device_address);
        return AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE;
    }

    DeviceVector *deviceVector;

    if (audio_is_output_device(device)) {
        deviceVector = &mAvailableOutputDevices;
    } else if (audio_is_input_device(device)) {
        deviceVector = &mAvailableInputDevices;
    } else {
        ALOGW("getDeviceConnectionState() invalid device type %08x", device);
        return AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE;
    }

    return (deviceVector->getDevice(device, String8(device_address)) != 0) ?
            AUDIO_POLICY_DEVICE_STATE_AVAILABLE : AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE;
}

void AudioPolicyManager::updateCallRouting(audio_devices_t rxDevice, int delayMs)
{
    bool createTxPatch = false;
    status_t status;
    audio_patch_handle_t afPatchHandle;
    DeviceVector deviceList;

    if(!hasPrimaryOutput()) {
        return;
    }
    audio_devices_t txDevice = getDeviceAndMixForInputSource(AUDIO_SOURCE_VOICE_COMMUNICATION);
    ALOGV("updateCallRouting device rxDevice %08x txDevice %08x", rxDevice, txDevice);

    // release existing RX patch if any
    if (mCallRxPatch != 0) {
        mpClientInterface->releaseAudioPatch(mCallRxPatch->mAfPatchHandle, 0);
        mCallRxPatch.clear();
    }
    // release TX patch if any
    if (mCallTxPatch != 0) {
        mpClientInterface->releaseAudioPatch(mCallTxPatch->mAfPatchHandle, 0);
        mCallTxPatch.clear();
    }

    // If the RX device is on the primary HW module, then use legacy routing method for voice calls
    // via setOutputDevice() on primary output.
    // Otherwise, create two audio patches for TX and RX path.
    if (availablePrimaryOutputDevices() & rxDevice) {
        setOutputDevice(mPrimaryOutput, rxDevice, true, delayMs);
        // If the TX device is also on the primary HW module, setOutputDevice() will take care
        // of it due to legacy implementation. If not, create a patch.
        if ((availablePrimaryInputDevices() & txDevice & ~AUDIO_DEVICE_BIT_IN)
                == AUDIO_DEVICE_NONE) {
            createTxPatch = true;
        }
    } else { // create RX path audio patch
        struct audio_patch patch;

        patch.num_sources = 1;
        patch.num_sinks = 1;
        deviceList = mAvailableOutputDevices.getDevicesFromType(rxDevice);
        ALOG_ASSERT(!deviceList.isEmpty(),
                    "updateCallRouting() selected device not in output device list");
        sp<DeviceDescriptor> rxSinkDeviceDesc = deviceList.itemAt(0);
        deviceList = mAvailableInputDevices.getDevicesFromType(AUDIO_DEVICE_IN_TELEPHONY_RX);
        ALOG_ASSERT(!deviceList.isEmpty(),
                    "updateCallRouting() no telephony RX device");
        sp<DeviceDescriptor> rxSourceDeviceDesc = deviceList.itemAt(0);

        rxSourceDeviceDesc->toAudioPortConfig(&patch.sources[0]);
        rxSinkDeviceDesc->toAudioPortConfig(&patch.sinks[0]);

        // request to reuse existing output stream if one is already opened to reach the RX device
        SortedVector<audio_io_handle_t> outputs =
                                getOutputsForDevice(rxDevice, mOutputs);
        audio_io_handle_t output = selectOutput(outputs,
                                                AUDIO_OUTPUT_FLAG_NONE,
                                                AUDIO_FORMAT_INVALID);
        if (output != AUDIO_IO_HANDLE_NONE) {
            sp<SwAudioOutputDescriptor> outputDesc = mOutputs.valueFor(output);
            ALOG_ASSERT(!outputDesc->isDuplicated(),
                        "updateCallRouting() RX device output is duplicated");
            outputDesc->toAudioPortConfig(&patch.sources[1]);
            patch.sources[1].ext.mix.usecase.stream = AUDIO_STREAM_PATCH;
            patch.num_sources = 2;
        }

        afPatchHandle = AUDIO_PATCH_HANDLE_NONE;
        status = mpClientInterface->createAudioPatch(&patch, &afPatchHandle, 0);
        ALOGW_IF(status != NO_ERROR, "updateCallRouting() error %d creating RX audio patch",
                                               status);
        if (status == NO_ERROR) {
            mCallRxPatch = new AudioPatch(&patch, mUidCached);
            mCallRxPatch->mAfPatchHandle = afPatchHandle;
            mCallRxPatch->mUid = mUidCached;
        }
        createTxPatch = true;
    }
    if (createTxPatch) { // create TX path audio patch
        struct audio_patch patch;

        patch.num_sources = 1;
        patch.num_sinks = 1;
        deviceList = mAvailableInputDevices.getDevicesFromType(txDevice);
        ALOG_ASSERT(!deviceList.isEmpty(),
                    "updateCallRouting() selected device not in input device list");
        sp<DeviceDescriptor> txSourceDeviceDesc = deviceList.itemAt(0);
        txSourceDeviceDesc->toAudioPortConfig(&patch.sources[0]);
        deviceList = mAvailableOutputDevices.getDevicesFromType(AUDIO_DEVICE_OUT_TELEPHONY_TX);
        ALOG_ASSERT(!deviceList.isEmpty(),
                    "updateCallRouting() no telephony TX device");
        sp<DeviceDescriptor> txSinkDeviceDesc = deviceList.itemAt(0);
        txSinkDeviceDesc->toAudioPortConfig(&patch.sinks[0]);

        SortedVector<audio_io_handle_t> outputs =
                                getOutputsForDevice(AUDIO_DEVICE_OUT_TELEPHONY_TX, mOutputs);
        audio_io_handle_t output = selectOutput(outputs,
                                                AUDIO_OUTPUT_FLAG_NONE,
                                                AUDIO_FORMAT_INVALID);
        // request to reuse existing output stream if one is already opened to reach the TX
        // path output device
        if (output != AUDIO_IO_HANDLE_NONE) {
            sp<AudioOutputDescriptor> outputDesc = mOutputs.valueFor(output);
            ALOG_ASSERT(!outputDesc->isDuplicated(),
                        "updateCallRouting() RX device output is duplicated");
            outputDesc->toAudioPortConfig(&patch.sources[1]);
            patch.sources[1].ext.mix.usecase.stream = AUDIO_STREAM_PATCH;
            patch.num_sources = 2;
        }

        // terminate active capture if on the same HW module as the call TX source device
        // FIXME: would be better to refine to only inputs whose profile connects to the
        // call TX device but this information is not in the audio patch and logic here must be
        // symmetric to the one in startInput()
        audio_io_handle_t activeInput = mInputs.getActiveInput();
        if (activeInput != 0) {
            sp<AudioInputDescriptor> activeDesc = mInputs.valueFor(activeInput);
            if (activeDesc->getModuleHandle() == txSourceDeviceDesc->getModuleHandle()) {
                //FIXME: consider all active sessions
                AudioSessionCollection activeSessions = activeDesc->getActiveAudioSessions();
                audio_session_t activeSession = activeSessions.keyAt(0);
                stopInput(activeInput, activeSession);
                releaseInput(activeInput, activeSession);
            }
        }

        afPatchHandle = AUDIO_PATCH_HANDLE_NONE;
        status = mpClientInterface->createAudioPatch(&patch, &afPatchHandle, 0);
        ALOGW_IF(status != NO_ERROR, "setPhoneState() error %d creating TX audio patch",
                                               status);
        if (status == NO_ERROR) {
            mCallTxPatch = new AudioPatch(&patch, mUidCached);
            mCallTxPatch->mAfPatchHandle = afPatchHandle;
            mCallTxPatch->mUid = mUidCached;
        }
    }
}

void AudioPolicyManager::setPhoneState(audio_mode_t state)
{
    ALOGV("setPhoneState() state %d", state);
    // store previous phone state for management of sonification strategy below
    int oldState = mEngine->getPhoneState();

    if (mEngine->setPhoneState(state) != NO_ERROR) {
        ALOGW("setPhoneState() invalid or same state %d", state);
        return;
    }
    /// Opens: can these line be executed after the switch of volume curves???
    // if leaving call state, handle special case of active streams
    // pertaining to sonification strategy see handleIncallSonification()
    if (isStateInCall(oldState)) {
        ALOGV("setPhoneState() in call state management: new state is %d", state);
        for (int stream = 0; stream < AUDIO_STREAM_FOR_POLICY_CNT; stream++) {
            handleIncallSonification((audio_stream_type_t)stream, false, true);
        }

        // force reevaluating accessibility routing when call stops
        mpClientInterface->invalidateStream(AUDIO_STREAM_ACCESSIBILITY);
    }

    /**
     * Switching to or from incall state or switching between telephony and VoIP lead to force
     * routing command.
     */
    bool force = ((is_state_in_call(oldState) != is_state_in_call(state))
                  || (is_state_in_call(state) && (state != oldState)));

    // check for device and output changes triggered by new phone state
    checkA2dpSuspend();
    checkOutputForAllStrategies();
    updateDevicesAndOutputs();

    int delayMs = 0;
    if (isStateInCall(state)) {
        nsecs_t sysTime = systemTime();
        for (size_t i = 0; i < mOutputs.size(); i++) {
            sp<SwAudioOutputDescriptor> desc = mOutputs.valueAt(i);
            // mute media and sonification strategies and delay device switch by the largest
            // latency of any output where either strategy is active.
            // This avoid sending the ring tone or music tail into the earpiece or headset.
#ifdef MTK_AUDIO //It should be real latency to avoid sound tail, not always using 5 secs. Android Issue
            if ((isStrategyActive(desc, STRATEGY_MEDIA,
                                  desc->latency()*2,
                                  sysTime) ||
                 isStrategyActive(desc, STRATEGY_SONIFICATION,
                                  desc->latency()*2,
                                  sysTime)) &&
                    (delayMs < (int)desc->latency()*2)) {
                delayMs = desc->mLatency*2;
            }
#else
            if ((isStrategyActive(desc, STRATEGY_MEDIA,
                                  SONIFICATION_HEADSET_MUSIC_DELAY,
                                  sysTime) ||
                 isStrategyActive(desc, STRATEGY_SONIFICATION,
                                  SONIFICATION_HEADSET_MUSIC_DELAY,
                                  sysTime)) &&
                    (delayMs < (int)desc->latency()*2)) {
                delayMs = desc->latency()*2;
            }
#endif
            setStrategyMute(STRATEGY_MEDIA, true, desc);
            setStrategyMute(STRATEGY_MEDIA, false, desc, MUTE_TIME_MS,
                getDeviceForStrategy(STRATEGY_MEDIA, true /*fromCache*/));
            setStrategyMute(STRATEGY_SONIFICATION, true, desc);
            setStrategyMute(STRATEGY_SONIFICATION, false, desc, MUTE_TIME_MS,
                getDeviceForStrategy(STRATEGY_SONIFICATION, true /*fromCache*/));
        }
    }

    if (hasPrimaryOutput()) {
        // Note that despite the fact that getNewOutputDevice() is called on the primary output,
        // the device returned is not necessarily reachable via this output
#ifdef MTK_AUDIO
        audio_devices_t rxDevice = getNewOutputDevice(mPrimaryOutput, false /*fromCache*/, true);
#else
        audio_devices_t rxDevice = getNewOutputDevice(mPrimaryOutput, false /*fromCache*/);
#endif
        // force routing command to audio hardware when ending call
        // even if no device change is needed
#ifdef MTK_AUDIO
        ALOGV("setPhoneState() rxDevice 0x%x pri.dev 0x%x", rxDevice,mPrimaryOutput->device());
#endif
        if (isStateInCall(oldState) && rxDevice == AUDIO_DEVICE_NONE) {
            rxDevice = mPrimaryOutput->device();
        }
#ifdef MTK_AUDIO
        if (mAudioPolicyVendorControl.isStateInCallOnly(state)) {
            updateCallRouting(rxDevice, delayMs);
        } else if (mAudioPolicyVendorControl.isStateInCallOnly(oldState))
#else

        if (state == AUDIO_MODE_IN_CALL) {
            updateCallRouting(rxDevice, delayMs);
        } else if (oldState == AUDIO_MODE_IN_CALL)
#endif
        {
            if (mCallRxPatch != 0) {
                mpClientInterface->releaseAudioPatch(mCallRxPatch->mAfPatchHandle, 0);
                mCallRxPatch.clear();
            }
            if (mCallTxPatch != 0) {
                mpClientInterface->releaseAudioPatch(mCallTxPatch->mAfPatchHandle, 0);
                mCallTxPatch.clear();
            }
            setOutputDevice(mPrimaryOutput, rxDevice, force, 0);
        } else {
            setOutputDevice(mPrimaryOutput, rxDevice, force, 0);
        }
    }
    // if entering in call state, handle special case of active streams
    // pertaining to sonification strategy see handleIncallSonification()
    if (isStateInCall(state)) {
        ALOGV("setPhoneState() in call state management: new state is %d", state);
        for (int stream = 0; stream < AUDIO_STREAM_FOR_POLICY_CNT; stream++) {
            handleIncallSonification((audio_stream_type_t)stream, true, true);
        }

        // force reevaluating accessibility routing when call starts
        mpClientInterface->invalidateStream(AUDIO_STREAM_ACCESSIBILITY);
    }

    // Flag that ringtone volume must be limited to music volume until we exit MODE_RINGTONE
    if (state == AUDIO_MODE_RINGTONE &&
        isStreamActive(AUDIO_STREAM_MUSIC, SONIFICATION_HEADSET_MUSIC_DELAY)) {
        mLimitRingtoneVolume = true;
    } else {
        mLimitRingtoneVolume = false;
    }
#ifdef MTK_AUDIO    // Mute Enforced_Audible when in Call
    if (!isStateInCall(oldState) && isStateInCall(state)) {
        setStreamMute(AUDIO_STREAM_ENFORCED_AUDIBLE, true, mPrimaryOutput);
    }
    else if (isStateInCall(oldState) && !isStateInCall(state)) {
        setStreamMute(AUDIO_STREAM_ENFORCED_AUDIBLE, false, mPrimaryOutput);
    }
#endif

}

audio_mode_t AudioPolicyManager::getPhoneState() {
    return mEngine->getPhoneState();
}

void AudioPolicyManager::setForceUse(audio_policy_force_use_t usage,
                                         audio_policy_forced_cfg_t config)
{
    ALOGV("setForceUse() usage %d, config %d, mPhoneState %d", usage, config, mEngine->getPhoneState());

    if (mEngine->setForceUse(usage, config) != NO_ERROR) {
        ALOGW("setForceUse() could not set force cfg %d for usage %d", config, usage);
        return;
    }
#ifdef MTK_CROSSMOUNT_SUPPORT
    if (mAudioPolicyVendorControl.getNeedResetInput()) {
        for (size_t i = 0; i < mInputs.size(); i++) {
        const sp<AudioInputDescriptor>  input_descriptor = mInputs.valueAt(i);
        if ((input_descriptor->isActive())
                && ((input_descriptor->mDevice & AUDIO_DEVICE_IN_REMOTE_SUBMIX) == AUDIO_DEVICE_IN_REMOTE_SUBMIX)
                && (input_descriptor->mPolicyMix != NULL)
                && (input_descriptor->mPolicyMix->mDeviceAddress == String8("CrossMount-Mic"))) {
                // When disconnect crossmount, disconnect remote-submix-out for normal routing to primary output
                // And this should happen before checkOutputForAllStrategies
                ALOGD("Stop remote submix in record");
                AudioSessionCollection activeSessions = input_descriptor->getActiveAudioSessions();
                audio_session_t activeSession = activeSessions.keyAt(0);
                stopInput(mInputs.keyAt(i), activeSession);
                releaseInput(mInputs.keyAt(i), activeSession);
                break;
            }
        }
    } else if (mAudioPolicyVendorControl.getStart2CrossMount()) {
        for (size_t i = 0; i < mInputs.size(); i++) {
            const sp<AudioInputDescriptor>  input_descriptor = mInputs.valueAt(i);
            if ((input_descriptor->isActive())
               && ((input_descriptor->mDevice & AUDIO_DEVICE_IN_REMOTE_SUBMIX) != AUDIO_DEVICE_IN_REMOTE_SUBMIX)) {
                    // When connect crossmount, disconnect
                    ALOGD("Start remote submix in record");
                    AudioSessionCollection activeSessions = input_descriptor->getActiveAudioSessions();
                    audio_session_t activeSession = activeSessions.keyAt(0);
                    stopInput(mInputs.keyAt(i), activeSession);
                    releaseInput(mInputs.keyAt(i), activeSession);
                    break;
                }
        }
    }
#endif
    bool forceVolumeReeval = (usage == AUDIO_POLICY_FORCE_FOR_COMMUNICATION) ||
            (usage == AUDIO_POLICY_FORCE_FOR_DOCK) ||
            (usage == AUDIO_POLICY_FORCE_FOR_SYSTEM);

    // check for device and output changes triggered by new force usage
    checkA2dpSuspend();
    checkOutputForAllStrategies();
    updateDevicesAndOutputs();

#ifdef MTK_AUDIO
    if (mAudioPolicyVendorControl.isStateInCallOnly(mEngine->getPhoneState()) && hasPrimaryOutput())
#else
    if (mEngine->getPhoneState() == AUDIO_MODE_IN_CALL && hasPrimaryOutput())
#endif
    {
        audio_devices_t newDevice = getNewOutputDevice(mPrimaryOutput, true /*fromCache*/);
        updateCallRouting(newDevice);
    }
    for (size_t i = 0; i < mOutputs.size(); i++) {
        sp<SwAudioOutputDescriptor> outputDesc = mOutputs.valueAt(i);
#ifdef MTK_AUDIO
        audio_devices_t newDevice = getNewOutputDevice(outputDesc, true /*fromCache*/, true);
        if ((!mAudioPolicyVendorControl.isStateInCallOnly(mEngine->getPhoneState())) || (outputDesc != mPrimaryOutput))
#else
        audio_devices_t newDevice = getNewOutputDevice(outputDesc, true /*fromCache*/);
        if ((mEngine->getPhoneState() != AUDIO_MODE_IN_CALL) || (outputDesc != mPrimaryOutput))
#endif
        {

            setOutputDevice(outputDesc, newDevice, (newDevice != AUDIO_DEVICE_NONE));
        }
        if (forceVolumeReeval && (newDevice != AUDIO_DEVICE_NONE)) {
            applyStreamVolumes(outputDesc, newDevice, 0, true);
        }
    }

    audio_io_handle_t activeInput = mInputs.getActiveInput();
    if (activeInput != 0) {
        sp<AudioInputDescriptor> activeDesc = mInputs.valueFor(activeInput);
        audio_devices_t newDevice = getNewInputDevice(activeInput);
        // Force new input selection if the new device can not be reached via current input
        if (activeDesc->mProfile->getSupportedDevices().types() & (newDevice & ~AUDIO_DEVICE_BIT_IN)) {
            setInputDevice(activeInput, newDevice);
        } else {
            closeInput(activeInput);
        }
    }
}

void AudioPolicyManager::setSystemProperty(const char* property, const char* value)
{
    ALOGV("setSystemProperty() property %s, value %s", property, value);
}

// Find a direct output profile compatible with the parameters passed, even if the input flags do
// not explicitly request a direct output
sp<IOProfile> AudioPolicyManager::getProfileForDirectOutput(
                                                               audio_devices_t device,
                                                               uint32_t samplingRate,
                                                               audio_format_t format,
                                                               audio_channel_mask_t channelMask,
                                                               audio_output_flags_t flags)
{
    // only retain flags that will drive the direct output profile selection
    // if explicitly requested
    static const uint32_t kRelevantFlags =
            (AUDIO_OUTPUT_FLAG_HW_AV_SYNC | AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD);
    flags =
        (audio_output_flags_t)((flags & kRelevantFlags) | AUDIO_OUTPUT_FLAG_DIRECT);

    sp<IOProfile> profile;

    for (size_t i = 0; i < mHwModules.size(); i++) {
        if (mHwModules[i]->mHandle == 0) {
            continue;
        }
        for (size_t j = 0; j < mHwModules[i]->mOutputProfiles.size(); j++) {
            sp<IOProfile> curProfile = mHwModules[i]->mOutputProfiles[j];
            if (!curProfile->isCompatibleProfile(device, String8(""),
                    samplingRate, NULL /*updatedSamplingRate*/,
                    format, NULL /*updatedFormat*/,
                    channelMask, NULL /*updatedChannelMask*/,
                    flags)) {
                continue;
            }
            // reject profiles not corresponding to a device currently available
            if ((mAvailableOutputDevices.types() & curProfile->getSupportedDevicesType()) == 0) {
                continue;
            }
            // if several profiles are compatible, give priority to one with offload capability
            if (profile != 0 && ((curProfile->getFlags() & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) == 0)) {
                continue;
            }
            profile = curProfile;
            if ((profile->getFlags() & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) != 0) {
                break;
            }
        }
    }
    return profile;
}

audio_io_handle_t AudioPolicyManager::getOutput(audio_stream_type_t stream,
                                                uint32_t samplingRate,
                                                audio_format_t format,
                                                audio_channel_mask_t channelMask,
                                                audio_output_flags_t flags,
                                                const audio_offload_info_t *offloadInfo)
{
    routing_strategy strategy = getStrategy(stream);
#ifdef MTK_CROSSMOUNT_SUPPORT
    audio_devices_t device = getDeviceForStrategy(strategy, false /*fromCache*/, flags);
#else
    audio_devices_t device = getDeviceForStrategy(strategy, false /*fromCache*/);
#endif
    ALOGV("getOutput() device %d, stream %d, samplingRate %d, format %x, channelMask %x, flags %x",
          device, stream, samplingRate, format, channelMask, flags);

    return getOutputForDevice(device, AUDIO_SESSION_ALLOCATE,
                              stream, samplingRate,format, channelMask,
                              flags, offloadInfo);
}

status_t AudioPolicyManager::getOutputForAttr(const audio_attributes_t *attr,
                                              audio_io_handle_t *output,
                                              audio_session_t session,
                                              audio_stream_type_t *stream,
                                              uid_t uid,
                                              uint32_t samplingRate,
                                              audio_format_t format,
                                              audio_channel_mask_t channelMask,
                                              audio_output_flags_t flags,
                                              audio_port_handle_t selectedDeviceId,
                                              const audio_offload_info_t *offloadInfo)
{
    audio_attributes_t attributes;
    if (attr != NULL) {
        if (!isValidAttributes(attr)) {
            ALOGE("getOutputForAttr() invalid attributes: usage=%d content=%d flags=0x%x tags=[%s]",
                  attr->usage, attr->content_type, attr->flags,
                  attr->tags);
            return BAD_VALUE;
        }
        attributes = *attr;
    } else {
        if (*stream < AUDIO_STREAM_MIN || *stream >= AUDIO_STREAM_PUBLIC_CNT) {
            ALOGE("getOutputForAttr():  invalid stream type");
            return BAD_VALUE;
        }
        stream_type_to_audio_attributes(*stream, &attributes);
    }
    sp<SwAudioOutputDescriptor> desc;
    if (mPolicyMixes.getOutputForAttr(attributes, uid, desc) == NO_ERROR) {
        ALOG_ASSERT(desc != 0, "Invalid desc returned by getOutputForAttr");
        if (!audio_has_proportional_frames(format)) {
            return BAD_VALUE;
        }
        *stream = streamTypefromAttributesInt(&attributes);
        *output = desc->mIoHandle;
        ALOGV("getOutputForAttr() returns output %d", *output);
        return NO_ERROR;
    }
    if (attributes.usage == AUDIO_USAGE_VIRTUAL_SOURCE) {
        ALOGW("getOutputForAttr() no policy mix found for usage AUDIO_USAGE_VIRTUAL_SOURCE");
        return BAD_VALUE;
    }

    ALOGV("getOutputForAttr() usage=%d, content=%d, tag=%s flags=%08x"
            " session %d selectedDeviceId %d",
            attributes.usage, attributes.content_type, attributes.tags, attributes.flags,
            session, selectedDeviceId);

    *stream = streamTypefromAttributesInt(&attributes);

    // Explicit routing?
    sp<DeviceDescriptor> deviceDesc;
    for (size_t i = 0; i < mAvailableOutputDevices.size(); i++) {
        if (mAvailableOutputDevices[i]->getId() == selectedDeviceId) {
            deviceDesc = mAvailableOutputDevices[i];
            break;
        }
    }
    mOutputRoutes.addRoute(session, *stream, SessionRoute::SOURCE_TYPE_NA, deviceDesc, uid);

    routing_strategy strategy = (routing_strategy) getStrategyForAttr(&attributes);
#ifdef MTK_CROSSMOUNT_SUPPORT
    audio_devices_t device = getDeviceForStrategy(strategy, false /*fromCache*/, flags);
#else
    audio_devices_t device = getDeviceForStrategy(strategy, false /*fromCache*/);
#endif

    if ((attributes.flags & AUDIO_FLAG_HW_AV_SYNC) != 0) {
        flags = (audio_output_flags_t)(flags | AUDIO_OUTPUT_FLAG_HW_AV_SYNC);
    }

    ALOGV("getOutputForAttr() device 0x%x, samplingRate %d, format %x, channelMask %x, flags %x",
          device, samplingRate, format, channelMask, flags);

    *output = getOutputForDevice(device, session, *stream,
                                 samplingRate, format, channelMask,
                                 flags, offloadInfo);
    if (*output == AUDIO_IO_HANDLE_NONE) {
        mOutputRoutes.removeRoute(session);
        return INVALID_OPERATION;
    }

    return NO_ERROR;
}

audio_io_handle_t AudioPolicyManager::getOutputForDevice(
        audio_devices_t device,
        audio_session_t session __unused,
        audio_stream_type_t stream,
        uint32_t samplingRate,
        audio_format_t format,
        audio_channel_mask_t channelMask,
        audio_output_flags_t flags,
        const audio_offload_info_t *offloadInfo)
{
    audio_io_handle_t output = AUDIO_IO_HANDLE_NONE;
    uint32_t latency = 0;
    status_t status;

#ifdef AUDIO_POLICY_TEST
    if (mCurOutput != 0) {
        ALOGV("getOutput() test output mCurOutput %d, samplingRate %d, format %d, channelMask %x, mDirectOutput %d",
                mCurOutput, mTestSamplingRate, mTestFormat, mTestChannels, mDirectOutput);

        if (mTestOutputs[mCurOutput] == 0) {
            ALOGV("getOutput() opening test output");
            sp<AudioOutputDescriptor> outputDesc = new SwAudioOutputDescriptor(NULL,
                                                                               mpClientInterface);
            outputDesc->mDevice = mTestDevice;
            outputDesc->mLatency = mTestLatencyMs;
            outputDesc->mFlags =
                    (audio_output_flags_t)(mDirectOutput ? AUDIO_OUTPUT_FLAG_DIRECT : 0);
            outputDesc->mRefCount[stream] = 0;
            audio_config_t config = AUDIO_CONFIG_INITIALIZER;
            config.sample_rate = mTestSamplingRate;
            config.channel_mask = mTestChannels;
            config.format = mTestFormat;
            if (offloadInfo != NULL) {
                config.offload_info = *offloadInfo;
            }
            status = mpClientInterface->openOutput(0,
                                                  &mTestOutputs[mCurOutput],
                                                  &config,
                                                  &outputDesc->mDevice,
                                                  String8(""),
                                                  &outputDesc->mLatency,
                                                  outputDesc->mFlags);
            if (status == NO_ERROR) {
                outputDesc->mSamplingRate = config.sample_rate;
                outputDesc->mFormat = config.format;
                outputDesc->mChannelMask = config.channel_mask;
                AudioParameter outputCmd = AudioParameter();
                outputCmd.addInt(String8("set_id"),mCurOutput);
                mpClientInterface->setParameters(mTestOutputs[mCurOutput],outputCmd.toString());
                addOutput(mTestOutputs[mCurOutput], outputDesc);
            }
        }
        return mTestOutputs[mCurOutput];
    }
#endif //AUDIO_POLICY_TEST

    // open a direct output if required by specified parameters
    //force direct flag if offload flag is set: offloading implies a direct output stream
    // and all common behaviors are driven by checking only the direct flag
    // this should normally be set appropriately in the policy configuration file
    if ((flags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) != 0) {
        flags = (audio_output_flags_t)(flags | AUDIO_OUTPUT_FLAG_DIRECT);
    }
    if ((flags & AUDIO_OUTPUT_FLAG_HW_AV_SYNC) != 0) {
        flags = (audio_output_flags_t)(flags | AUDIO_OUTPUT_FLAG_DIRECT);
    }
    // only allow deep buffering for music stream type
    if (stream != AUDIO_STREAM_MUSIC) {
        flags = (audio_output_flags_t)(flags &~AUDIO_OUTPUT_FLAG_DEEP_BUFFER);
    } else if (/* stream == AUDIO_STREAM_MUSIC && */
            flags == AUDIO_OUTPUT_FLAG_NONE &&
            property_get_bool("audio.deep_buffer.media", false /* default_value */)) {
        // use DEEP_BUFFER as default output for music stream type
        flags = (audio_output_flags_t)AUDIO_OUTPUT_FLAG_DEEP_BUFFER;
    }
    if (stream == AUDIO_STREAM_TTS) {
        flags = AUDIO_OUTPUT_FLAG_TTS;
    }

    sp<IOProfile> profile;

    // skip direct output selection if the request can obviously be attached to a mixed output
    // and not explicitly requested
    if (((flags & AUDIO_OUTPUT_FLAG_DIRECT) == 0) &&
            audio_is_linear_pcm(format) && samplingRate <= SAMPLE_RATE_HZ_MAX &&
            audio_channel_count_from_out_mask(channelMask) <= 2) {
        goto non_direct_output;
    }
#ifdef MTK_AUDIO
    if (device == AUDIO_DEVICE_OUT_AUX_DIGITAL  && (flags & AUDIO_OUTPUT_FLAG_DIRECT) == 0 &&
        audio_is_linear_pcm(format)) {
        ALOGD("HDMI skip direct output");
        goto non_direct_output;
    }
#endif

    // Do not allow offloading if one non offloadable effect is enabled or MasterMono is enabled.
    // This prevents creating an offloaded track and tearing it down immediately after start
    // when audioflinger detects there is an active non offloadable effect.
    // FIXME: We should check the audio session here but we do not have it in this context.
    // This may prevent offloading in rare situations where effects are left active by apps
    // in the background.

    if (((flags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) == 0) ||
            !(mEffects.isNonOffloadableEffectEnabled() || mMasterMono)) {
        profile = getProfileForDirectOutput(device,
                                           samplingRate,
                                           format,
                                           channelMask,
                                           (audio_output_flags_t)flags);
    }

    if (profile != 0) {
        sp<SwAudioOutputDescriptor> outputDesc = NULL;
#ifdef MTK_AUDIO  // for offload audio case only
        if (flags&AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) {
            audio_io_handle_t primary_output = mPrimaryOutput->mIoHandle;
            ALOGD("pcm output:%x", primary_output);
            // if pcm is writing mute data, let it standby
            if (primary_output != 0) {
                String8 ret = mpClientInterface->getParameters(primary_output,
                                        String8(AudioParameter::keyOffloadAudioCheckSupport));
                if (ret == String8("OffloadAudio_Check_Support=no")) {
                    ALOGD("offload is not support, reject");
                    return 0;
                }
                ret = mpClientInterface->getParameters(primary_output,
                                        String8(AudioParameter::keyOffloadAudioDoStandybyWhenMute));
                ALOGD("getOutput offload getParameter %s=%s",
                                        AudioParameter::keyOffloadAudioDoStandybyWhenMute, ret.string());
                if (ret == String8("yes")) {
                    ALOGD("pcm is standby because offload join");
                }
                audio_io_handle_t offload_output = getOffloadOutput();
                ALOGD("offload output status:%d", offload_output);
                if (offload_output != 0) {
                    ALOGD("offload is reject due to there is an offload playing");
                    return 0;
                }
            }
        }
#endif

        for (size_t i = 0; i < mOutputs.size(); i++) {
            sp<SwAudioOutputDescriptor> desc = mOutputs.valueAt(i);
            if (!desc->isDuplicated() && (profile == desc->mProfile)) {
                outputDesc = desc;
                // reuse direct output if currently open and configured with same parameters
                if ((samplingRate == outputDesc->mSamplingRate) &&
                        audio_formats_match(format, outputDesc->mFormat) &&
                        (channelMask == outputDesc->mChannelMask)) {
                    outputDesc->mDirectOpenCount++;
                    ALOGV("getOutput() reusing direct output %d", mOutputs.keyAt(i));
                    return mOutputs.keyAt(i);
                }
            }
        }
        // close direct output if currently open and configured with different parameters
        if (outputDesc != NULL) {
            closeOutput(outputDesc->mIoHandle);
        }

        // if the selected profile is offloaded and no offload info was specified,
        // create a default one
        audio_offload_info_t defaultOffloadInfo = AUDIO_INFO_INITIALIZER;
        if ((profile->getFlags() & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) && !offloadInfo) {
            flags = (audio_output_flags_t)(flags | AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD);
            defaultOffloadInfo.sample_rate = samplingRate;
            defaultOffloadInfo.channel_mask = channelMask;
            defaultOffloadInfo.format = format;
            defaultOffloadInfo.stream_type = stream;
            defaultOffloadInfo.bit_rate = 0;
            defaultOffloadInfo.duration_us = -1;
            defaultOffloadInfo.has_video = true; // conservative
            defaultOffloadInfo.is_streaming = true; // likely
            offloadInfo = &defaultOffloadInfo;
        }

        outputDesc = new SwAudioOutputDescriptor(profile, mpClientInterface);
        outputDesc->mDevice = device;
        outputDesc->mLatency = 0;
        outputDesc->mFlags = (audio_output_flags_t)(outputDesc->mFlags | flags);
        audio_config_t config = AUDIO_CONFIG_INITIALIZER;
        config.sample_rate = samplingRate;
        config.channel_mask = channelMask;
        config.format = format;
        if (offloadInfo != NULL) {
            config.offload_info = *offloadInfo;
        }
        status = mpClientInterface->openOutput(profile->getModuleHandle(),
                                               &output,
                                               &config,
                                               &outputDesc->mDevice,
                                               String8(""),
                                               &outputDesc->mLatency,
                                               outputDesc->mFlags);

        // only accept an output with the requested parameters
        if (status != NO_ERROR ||
#ifndef MTK_AUDIO_TUNNELING_SUPPORT
            (samplingRate != 0 && samplingRate != config.sample_rate) ||
#endif
            (format != AUDIO_FORMAT_DEFAULT && !audio_formats_match(format, config.format)) ||
            (channelMask != 0 && channelMask != config.channel_mask)) {
            ALOGV("getOutput() failed opening direct output: output %d samplingRate %d %d,"
                    "format %d %d, channelMask %04x %04x", output, samplingRate,
                    outputDesc->mSamplingRate, format, outputDesc->mFormat, channelMask,
                    outputDesc->mChannelMask);
            if (output != AUDIO_IO_HANDLE_NONE) {
                mpClientInterface->closeOutput(output);
            }
            // fall back to mixer output if possible when the direct output could not be open
            if (audio_is_linear_pcm(format) && samplingRate <= SAMPLE_RATE_HZ_MAX) {
                goto non_direct_output;
            }
            return AUDIO_IO_HANDLE_NONE;
        }
        outputDesc->mSamplingRate = config.sample_rate;
        outputDesc->mChannelMask = config.channel_mask;
        outputDesc->mFormat = config.format;
        outputDesc->mRefCount[stream] = 0;
        outputDesc->mStopTime[stream] = 0;
        outputDesc->mDirectOpenCount = 1;

        audio_io_handle_t srcOutput = getOutputForEffect();
        addOutput(output, outputDesc);
        audio_io_handle_t dstOutput = getOutputForEffect();
        if (dstOutput == output) {
#ifdef DOLBY_DAP
            status_t status = mpClientInterface->moveEffects(AUDIO_SESSION_OUTPUT_MIX, srcOutput, dstOutput);
            if (status == NO_ERROR) {
                for (size_t i = 0; i < mEffects.size(); i++) {
                    sp<EffectDescriptor> desc = mEffects.valueAt(i);
                    if (desc->mSession == AUDIO_SESSION_OUTPUT_MIX) {
                        // update the mIo member of EffectDescriptor for the global effect
                        ALOGV("%s updating mIo", __FUNCTION__);
                        desc->mIo = dstOutput;
                    }
                }
            } else {
                ALOGW("%s moveEffects from %d to %d failed", __FUNCTION__, srcOutput, dstOutput);
            }
#else // DOLBY_END
            mpClientInterface->moveEffects(AUDIO_SESSION_OUTPUT_MIX, srcOutput, dstOutput);
#endif
        }
        mPreviousOutputs = mOutputs;
        ALOGV("getOutput() returns new direct output %d", output);
        mpClientInterface->onAudioPortListUpdate();
        return output;
    }

non_direct_output:

    // A request for HW A/V sync cannot fallback to a mixed output because time
    // stamps are embedded in audio data
    if ((flags & AUDIO_OUTPUT_FLAG_HW_AV_SYNC) != 0) {
        return AUDIO_IO_HANDLE_NONE;
    }

    // ignoring channel mask due to downmix capability in mixer

    // open a non direct output

    // for non direct outputs, only PCM is supported
    if (audio_is_linear_pcm(format)) {
        // get which output is suitable for the specified stream. The actual
        // routing change will happen when startOutput() will be called
        SortedVector<audio_io_handle_t> outputs = getOutputsForDevice(device, mOutputs);

        // at this stage we should ignore the DIRECT flag as no direct output could be found earlier
        flags = (audio_output_flags_t)(flags & ~AUDIO_OUTPUT_FLAG_DIRECT);
        output = selectOutput(outputs, flags, format);
    }
    ALOGW_IF((output == 0), "getOutput() could not find output for stream %d, samplingRate %d,"
            "format %d, channels %x, flags %x", stream, samplingRate, format, channelMask, flags);

    ALOGV("  getOutputForDevice() returns output %d", output);

    return output;
}

audio_io_handle_t AudioPolicyManager::selectOutput(const SortedVector<audio_io_handle_t>& outputs,
                                                       audio_output_flags_t flags,
                                                       audio_format_t format)
{
    // select one output among several that provide a path to a particular device or set of
    // devices (the list was previously build by getOutputsForDevice()).
    // The priority is as follows:
    // 1: the output with the highest number of requested policy flags
    // 2: the output with the bit depth the closest to the requested one
    // 3: the primary output
    // 4: the first output in the list

    if (outputs.size() == 0) {
        return 0;
    }
    if (outputs.size() == 1) {
        return outputs[0];
    }

    int maxCommonFlags = 0;
    audio_io_handle_t outputForFlags = 0;
    audio_io_handle_t outputForPrimary = 0;
    audio_io_handle_t outputForFormat = 0;
    audio_format_t bestFormat = AUDIO_FORMAT_INVALID;
    audio_format_t bestFormatForFlags = AUDIO_FORMAT_INVALID;

    for (size_t i = 0; i < outputs.size(); i++) {
        sp<SwAudioOutputDescriptor> outputDesc = mOutputs.valueFor(outputs[i]);
        if (!outputDesc->isDuplicated()) {
            // if a valid format is specified, skip output if not compatible
            if (format != AUDIO_FORMAT_INVALID) {
                if (outputDesc->mFlags & AUDIO_OUTPUT_FLAG_DIRECT) {
                    if (!audio_formats_match(format, outputDesc->mFormat)) {
                        continue;
                    }
                } else if (!audio_is_linear_pcm(format)) {
                    continue;
                }
                if (AudioPort::isBetterFormatMatch(
                        outputDesc->mFormat, bestFormat, format)) {
                    outputForFormat = outputs[i];
                    bestFormat = outputDesc->mFormat;
                }
            }

            int commonFlags = popcount(outputDesc->mProfile->getFlags() & flags);
            if (commonFlags >= maxCommonFlags) {
                if (commonFlags == maxCommonFlags) {
                    if (AudioPort::isBetterFormatMatch(
                            outputDesc->mFormat, bestFormatForFlags, format)) {
                        outputForFlags = outputs[i];
                        bestFormatForFlags = outputDesc->mFormat;
                    }
                } else {
                    outputForFlags = outputs[i];
                    maxCommonFlags = commonFlags;
                    bestFormatForFlags = outputDesc->mFormat;
                }
#ifdef MTK_AUDIO
                MTK_ALOGV("selectOutput() commonFlags for output %d, %04x", outputs[i], commonFlags);
#else
                ALOGV("selectOutput() commonFlags for output %d, %04x", outputs[i], commonFlags);
#endif
            }
            if (outputDesc->mProfile->getFlags() & AUDIO_OUTPUT_FLAG_PRIMARY) {
                outputForPrimary = outputs[i];
            }
        }
    }

    if (outputForFlags != 0) {
        return outputForFlags;
    }
    if (outputForFormat != 0) {
        return outputForFormat;
    }
    if (outputForPrimary != 0) {
        return outputForPrimary;
    }

    return outputs[0];
}

status_t AudioPolicyManager::startOutput(audio_io_handle_t output,
                                             audio_stream_type_t stream,
                                             audio_session_t session)
{
    ALOGV("startOutput() output %d, stream %d, session %d",
          output, stream, session);
    ssize_t index = mOutputs.indexOfKey(output);
    if (index < 0) {
        ALOGW("startOutput() unknown output %d", output);
        return BAD_VALUE;
    }

    sp<SwAudioOutputDescriptor> outputDesc = mOutputs.valueAt(index);

    // Routing?
    mOutputRoutes.incRouteActivity(session);

    audio_devices_t newDevice;
    AudioMix *policyMix = NULL;
    const char *address = NULL;
    if (outputDesc->mPolicyMix != NULL) {
        policyMix = outputDesc->mPolicyMix;
        address = policyMix->mDeviceAddress.string();
        if ((policyMix->mRouteFlags & MIX_ROUTE_FLAG_RENDER) == MIX_ROUTE_FLAG_RENDER) {
            newDevice = policyMix->mDeviceType;
        } else {
            newDevice = AUDIO_DEVICE_OUT_REMOTE_SUBMIX;
        }
    }
#if defined(MTK_CROSSMOUNT_SUPPORT)
    else if ((mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_RECORD) == AUDIO_POLICY_FORCE_REMOTE_SUBMIX_IN_WITH_FLAGS && outputDesc->device()== AUDIO_DEVICE_OUT_REMOTE_SUBMIX) && !mAudioPolicyVendorControl.getCrossMountMicAudioMixerEnable()){
        ALOGD("CrossMount mForceUse[%d]=%d outputDesc->device() 0x%x",AUDIO_POLICY_FORCE_FOR_RECORD,mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_RECORD),outputDesc->device());
        newDevice = AUDIO_DEVICE_OUT_REMOTE_SUBMIX;
    }
#endif
    else if (mOutputRoutes.hasRouteChanged(session)) {
#ifdef MTK_AUDIO    // Suppose outputs which belong to the same module should refer each output status for routing
        newDevice = getNewOutputDevice(outputDesc, false /*fromCache*/, true);
#else
        newDevice = getNewOutputDevice(outputDesc, false /*fromCache*/);
#endif

        checkStrategyRoute(getStrategy(stream), output);
    } else {
        newDevice = AUDIO_DEVICE_NONE;
    }

    uint32_t delayMs = 0;

    status_t status = startSource(outputDesc, stream, newDevice, address, &delayMs);

    if (status != NO_ERROR) {
        mOutputRoutes.decRouteActivity(session);
        return status;
    }
    // Automatically enable the remote submix input when output is started on a re routing mix
    // of type MIX_TYPE_RECORDERS
    if (audio_is_remote_submix_device(newDevice) && policyMix != NULL &&
            policyMix->mMixType == MIX_TYPE_RECORDERS) {
            setDeviceConnectionStateInt(AUDIO_DEVICE_IN_REMOTE_SUBMIX,
                    AUDIO_POLICY_DEVICE_STATE_AVAILABLE,
                    address,
                    "remote-submix");
    }

    if (delayMs != 0) {
        usleep(delayMs * 1000);
    }

#ifdef DOLBY_UDC
    // It is observed that in some use-cases where both outputs are present eg. bluetooth and headphone,
    // the output for particular stream type is decided in this routine. Hence we must call
    // getDeviceForStrategy in order to get the current active output for this stream type and update
    // the dolby system property.
    if (stream == AUDIO_STREAM_MUSIC) {
        audio_devices_t audioOutputDevice = getDeviceForStrategy(getStrategy(AUDIO_STREAM_MUSIC), true);
        mDolbyAudioPolicy.setEndpointSystemProperty(audioOutputDevice, mHwModules);
    }
#endif // DOLBY_UDC
#ifdef DOLBY_DAP_MOVE_EFFECT
    // Note: The global effect can't be taken away from the deep-buffered output (source output) if there're still
    //       music playing on the deep-buffered output.
    sp<SwAudioOutputDescriptor> srcOutputDesc = mOutputs.valueFor(mDolbyAudioPolicy.output());
    if ((stream == AUDIO_STREAM_MUSIC) && mDolbyAudioPolicy.shouldMoveToOutput(output, outputDesc->mFlags) && srcOutputDesc != NULL &&
        ((srcOutputDesc->mFlags & AUDIO_OUTPUT_FLAG_DEEP_BUFFER) == 0 || srcOutputDesc->mRefCount[AUDIO_STREAM_MUSIC] == 0)) {
        status_t status = mpClientInterface->moveEffects(DOLBY_MOVE_EFFECT_SIGNAL, mDolbyAudioPolicy.output(), output);
        if (status == NO_ERROR) {
            mDolbyAudioPolicy.movedToOutput(output);
        }
    }
#endif // DOLBY_END
    return status;
}

status_t AudioPolicyManager::startSource(sp<AudioOutputDescriptor> outputDesc,
                                             audio_stream_type_t stream,
                                             audio_devices_t device,
                                             const char *address,
                                             uint32_t *delayMs)
{
    // cannot start playback of STREAM_TTS if any other output is being used
    uint32_t beaconMuteLatency = 0;

    *delayMs = 0;
    if (stream == AUDIO_STREAM_TTS) {
        ALOGV("\t found BEACON stream");
        if (!mTtsOutputAvailable && mOutputs.isAnyOutputActive(AUDIO_STREAM_TTS /*streamToIgnore*/)) {
            return INVALID_OPERATION;
        } else {
            beaconMuteLatency = handleEventForBeacon(STARTING_BEACON);
        }
    } else {
        // some playback other than beacon starts
        beaconMuteLatency = handleEventForBeacon(STARTING_OUTPUT);
#ifdef MTK_AUDIO  // ALPS02035392, delay startoutput performance
        if (!mOutputs.isAnyOutputActive(AUDIO_STREAM_TTS)) {
            beaconMuteLatency = 0;
        }
#endif
    }

    // check active before incrementing usage count
    bool force = !outputDesc->isActive();

#ifdef MTK_AUDIO
    // mOutputFirstAcitive = true -> output is not active before startSource, and skip mute strategy at setOutputDevice
    if (outputDesc->sharesHwModuleWith(mPrimaryOutput)) {
        outputDesc->mOutputFirstActive = !outputDesc->isActive() && !mPrimaryOutput->isActive();
    } else {
        outputDesc->mOutputFirstActive = !outputDesc->isActive();
    }
#endif
    // increment usage count for this stream on the requested output:
    // NOTE that the usage count is the same for duplicated output and hardware output which is
    // necessary for a correct control of hardware output routing by startOutput() and stopOutput()
    outputDesc->changeRefCount(stream, 1);

#if defined(MTK_AUDIO) && defined(SUPPORT_ANDROID_FM_PLAYER)
    if (outputDesc->mRefCount[stream] == 1 || device != AUDIO_DEVICE_NONE || (mFMDirectAudioPatchEnable && outputDesc->mRefCount[stream] == 2)) //ALPS02336335
#else
    if (outputDesc->mRefCount[stream] == 1 || device != AUDIO_DEVICE_NONE)
#endif
    {
        // starting an output being rerouted?
        if (device == AUDIO_DEVICE_NONE) {
#ifdef MTK_AUDIO  // Suppose outputs which belong to the same module should refer each output status for routing
            device = getNewOutputDevice(outputDesc, false /*fromCache*/, true);
#else
            device = getNewOutputDevice(outputDesc, false /*fromCache*/);
#endif
        }
        routing_strategy strategy = getStrategy(stream);
        bool shouldWait = (strategy == STRATEGY_SONIFICATION) ||
                            (strategy == STRATEGY_SONIFICATION_RESPECTFUL) ||
                            (beaconMuteLatency > 0);
        uint32_t waitMs = beaconMuteLatency;
        for (size_t i = 0; i < mOutputs.size(); i++) {
            sp<AudioOutputDescriptor> desc = mOutputs.valueAt(i);
            if (desc != outputDesc) {
                // force a device change if any other output is managed by the same hw
                // module and has a current device selection that differs from selected device.
                // In this case, the audio HAL must receive the new device selection so that it can
                // change the device currently selected by the other active output.
                if (outputDesc->sharesHwModuleWith(desc) &&
                    desc->device() != device) {
                    force = true;
                }
                // wait for audio on other active outputs to be presented when starting
                // a notification so that audio focus effect can propagate, or that a mute/unmute
                // event occurred for beacon
                uint32_t latency = desc->latency();
                if (shouldWait && desc->isActive(latency * 2) && (waitMs < latency)) {
                    waitMs = latency;
                }
            }
        }
        uint32_t muteWaitMs = setOutputDevice(outputDesc, device, force, 0, NULL, address);

        // handle special case for sonification while in call
        if (isInCall()) {
            handleIncallSonification(stream, true, false);
        }

#ifdef MTK_AUDIO // ALPS02462525 usb output will apply usb+spk volume, correct to apply usb only
        device = (audio_devices_t)(device & outputDesc->supportedDevices());
        if (device == AUDIO_DEVICE_NONE) {
            device = outputDesc->mDevice;
        }
#endif
        // apply volume rules for current stream and device if necessary
        checkAndSetVolume(stream,
                          mVolumeCurves->getVolumeIndex(stream, device),
                          outputDesc,
                          device);

        // update the outputs if starting an output with a stream that can affect notification
        // routing
        handleNotificationRoutingForStream(stream);

        // force reevaluating accessibility routing when ringtone or alarm starts
        if (strategy == STRATEGY_SONIFICATION) {
            mpClientInterface->invalidateStream(AUDIO_STREAM_ACCESSIBILITY);
        }
    }
    return NO_ERROR;
}


status_t AudioPolicyManager::stopOutput(audio_io_handle_t output,
                                            audio_stream_type_t stream,
                                            audio_session_t session)
{
    ALOGV("stopOutput() output %d, stream %d, session %d", output, stream, session);
    ssize_t index = mOutputs.indexOfKey(output);
    if (index < 0) {
        ALOGW("stopOutput() unknown output %d", output);
        return BAD_VALUE;
    }

    sp<SwAudioOutputDescriptor> outputDesc = mOutputs.valueAt(index);

    if (outputDesc->mRefCount[stream] == 1) {
        // Automatically disable the remote submix input when output is stopped on a
        // re routing mix of type MIX_TYPE_RECORDERS
        if (audio_is_remote_submix_device(outputDesc->mDevice) &&
                outputDesc->mPolicyMix != NULL &&
                outputDesc->mPolicyMix->mMixType == MIX_TYPE_RECORDERS) {
            setDeviceConnectionStateInt(AUDIO_DEVICE_IN_REMOTE_SUBMIX,
                    AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE,
                    outputDesc->mPolicyMix->mDeviceAddress,
                    "remote-submix");
        }
    }

    // Routing?
    bool forceDeviceUpdate = false;
    if (outputDesc->mRefCount[stream] > 0) {
        int activityCount = mOutputRoutes.decRouteActivity(session);
        forceDeviceUpdate = (mOutputRoutes.hasRoute(session) && (activityCount == 0));

        if (forceDeviceUpdate) {
            checkStrategyRoute(getStrategy(stream), AUDIO_IO_HANDLE_NONE);
        }
    }

    return stopSource(outputDesc, stream, forceDeviceUpdate);
}

status_t AudioPolicyManager::stopSource(sp<AudioOutputDescriptor> outputDesc,
                                            audio_stream_type_t stream,
                                            bool forceDeviceUpdate)
{
    // always handle stream stop, check which stream type is stopping
    handleEventForBeacon(stream == AUDIO_STREAM_TTS ? STOPPING_BEACON : STOPPING_OUTPUT);

    // handle special case for sonification while in call
    if (isInCall()) {
        handleIncallSonification(stream, false, false);
    }

    if (outputDesc->mRefCount[stream] > 0) {
        // decrement usage count of this stream on the output
        outputDesc->changeRefCount(stream, -1);

        // store time at which the stream was stopped - see isStreamActive()
        if (outputDesc->mRefCount[stream] == 0 || forceDeviceUpdate) {
            outputDesc->mStopTime[stream] = systemTime();
            audio_devices_t newDevice = getNewOutputDevice(outputDesc, false /*fromCache*/);
            // delay the device switch by twice the latency because stopOutput() is executed when
            // the track stop() command is received and at that time the audio track buffer can
            // still contain data that needs to be drained. The latency only covers the audio HAL
            // and kernel buffers. Also the latency does not always include additional delay in the
            // audio path (audio DSP, CODEC ...)
#if MTK_AUDIO
            audio_devices_t prevDevice = outputDesc->mDevice;
            setOutputDevice(outputDesc, newDevice, false, outputDesc->latency()*2);
#ifdef MTK_AUDIO_GAIN_TABLE
            if (prevDevice == newDevice) {
                applyStreamVolumes(outputDesc, newDevice, outputDesc->latency()*2); //gain table need apply analog volume after stopOutput
            }
#endif
#else
            setOutputDevice(outputDesc, newDevice, false, outputDesc->latency()*2);
#endif
#if MTK_AUDIO
            // ALPS02499755, newDevice should refer ShareHwModule
            newDevice = getNewOutputDevice(outputDesc, false /*fromCache*/, true);
#endif
            // force restoring the device selection on other active outputs if it differs from the
            // one being selected for this output
            for (size_t i = 0; i < mOutputs.size(); i++) {
                audio_io_handle_t curOutput = mOutputs.keyAt(i);
                sp<AudioOutputDescriptor> desc = mOutputs.valueAt(i);
                if (desc != outputDesc &&
                        desc->isActive() &&
#ifdef MTK_AUDIO        // ALPS02713965 skip rerouting when duplicated output stops for a stream type
                        !(outputDesc->isDuplicated() && outputDesc->isActive() && outputDesc->sharesHwModuleWith(desc)) &&
#endif
                        outputDesc->sharesHwModuleWith(desc) &&
                        (newDevice != desc->device()
#ifdef MTK_AUDIO // ALPS02626190, should reroute by active output
                        || newDevice != outputDesc->device()
#endif
                        )) {
                    setOutputDevice(desc,
                                    getNewOutputDevice(desc, false /*fromCache*/),
                                    true,
                                    outputDesc->latency()*2);
                }
            }
            // update the outputs if stopping one with a stream that can affect notification routing
            handleNotificationRoutingForStream(stream);
        }
        return NO_ERROR;
    } else {
        ALOGW("stopOutput() refcount is already 0");
        return INVALID_OPERATION;
    }
}

void AudioPolicyManager::releaseOutput(audio_io_handle_t output,
                                       audio_stream_type_t stream __unused,
                                       audio_session_t session __unused)
{
    ALOGV("releaseOutput() %d", output);
    ssize_t index = mOutputs.indexOfKey(output);
    if (index < 0) {
        ALOGW("releaseOutput() releasing unknown output %d", output);
        return;
    }

#ifdef AUDIO_POLICY_TEST
    int testIndex = testOutputIndex(output);
    if (testIndex != 0) {
        sp<AudioOutputDescriptor> outputDesc = mOutputs.valueAt(index);
        if (outputDesc->isActive()) {
            mpClientInterface->closeOutput(output);
            removeOutput(output);
            mTestOutputs[testIndex] = 0;
        }
        return;
    }
#endif //AUDIO_POLICY_TEST

    // Routing
    mOutputRoutes.removeRoute(session);

    sp<SwAudioOutputDescriptor> desc = mOutputs.valueAt(index);
    if (desc->mFlags & AUDIO_OUTPUT_FLAG_DIRECT) {
        if (desc->mDirectOpenCount <= 0) {
            ALOGW("releaseOutput() invalid open count %d for output %d",
                                                              desc->mDirectOpenCount, output);
            return;
        }
        if (--desc->mDirectOpenCount == 0) {
            closeOutput(output);
            // If effects where present on the output, audioflinger moved them to the primary
            // output by default: move them back to the appropriate output.
            audio_io_handle_t dstOutput = getOutputForEffect();
            if (hasPrimaryOutput() && dstOutput != mPrimaryOutput->mIoHandle) {
#ifdef DOLBY_DAP
                status_t status = mpClientInterface->moveEffects(AUDIO_SESSION_OUTPUT_MIX, mPrimaryOutput->mIoHandle, dstOutput);
                if (status == NO_ERROR) {
                    for (size_t i = 0; i < mEffects.size(); i++) {
                        sp<EffectDescriptor> desc = mEffects.valueAt(i);
                        if (desc->mSession == AUDIO_SESSION_OUTPUT_MIX) {
                            // update the mIo member of EffectDescriptor for the global effect
                            ALOGV("%s updating mIo", __FUNCTION__);
                            desc->mIo = dstOutput;
                        }
                    }
                } else {
                    ALOGW("%s moveEffects from %d to %d failed", __FUNCTION__, mPrimaryOutput->mIoHandle, dstOutput);
                }
#else // DOLBY_END
                mpClientInterface->moveEffects(AUDIO_SESSION_OUTPUT_MIX,
                                               mPrimaryOutput->mIoHandle, dstOutput);
#endif
            }
            mpClientInterface->onAudioPortListUpdate();
        }
    }
}


status_t AudioPolicyManager::getInputForAttr(const audio_attributes_t *attr,
                                             audio_io_handle_t *input,
                                             audio_session_t session,
                                             uid_t uid,
                                             uint32_t samplingRate,
                                             audio_format_t format,
                                             audio_channel_mask_t channelMask,
                                             audio_input_flags_t flags,
                                             audio_port_handle_t selectedDeviceId,
                                             input_type_t *inputType)
{
    ALOGV("getInputForAttr() source %d, samplingRate %d, format %d, channelMask %x,"
            "session %d, flags %#x",
          attr->source, samplingRate, format, channelMask, session, flags);

    *input = AUDIO_IO_HANDLE_NONE;
    *inputType = API_INPUT_INVALID;
    audio_devices_t device;
    // handle legacy remote submix case where the address was not always specified
    String8 address = String8("");
    audio_source_t inputSource = attr->source;
    audio_source_t halInputSource;
    AudioMix *policyMix = NULL;

    if (inputSource == AUDIO_SOURCE_DEFAULT) {
        inputSource = AUDIO_SOURCE_MIC;
    }
#ifdef MTK_CROSSMOUNT_SUPPORT
        audio_attributes_t stAttr;
        memcpy(&stAttr, attr, sizeof(audio_attributes_t));
        if ((mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_RECORD) == AUDIO_POLICY_FORCE_REMOTE_SUBMIX_IN_WITH_FLAGS ||
            mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_RECORD) == AUDIO_POLICY_FORCE_REMOTE_SUBMIX_IN) && (inputSource == AUDIO_SOURCE_MIC ||
            inputSource == AUDIO_SOURCE_CAMCORDER ||
            inputSource == AUDIO_SOURCE_VOICE_RECOGNITION ||
            inputSource == AUDIO_SOURCE_VOICE_COMMUNICATION ||
            inputSource == AUDIO_SOURCE_VOICE_UNLOCK ||
            inputSource == AUDIO_SOURCE_CUSTOMIZATION1 ||
            inputSource == AUDIO_SOURCE_CUSTOMIZATION2 ||
            inputSource == AUDIO_SOURCE_CUSTOMIZATION3 ||
            inputSource == AUDIO_SOURCE_HOTWORD) && mAudioPolicyVendorControl.getCrossMountMicAudioMixerEnable()) {
            ALOGV("Record change inputsrc 0x%x to inputsrc 0x%x", inputSource, AUDIO_SOURCE_REMOTE_SUBMIX);
            inputSource = AUDIO_SOURCE_REMOTE_SUBMIX;
            strcpy(stAttr.tags,"addr=CrossMount-Mic");
        }
        halInputSource = inputSource;

        // Explicit routing?
        sp<DeviceDescriptor> deviceDesc;
        for (size_t i = 0; i < mAvailableInputDevices.size(); i++) {
            if (mAvailableInputDevices[i]->getId() == selectedDeviceId) {
                deviceDesc = mAvailableInputDevices[i];
                break;
            }
        }
        mInputRoutes.addRoute(session, SessionRoute::STREAM_TYPE_NA, inputSource, deviceDesc, uid);

        if (inputSource == AUDIO_SOURCE_REMOTE_SUBMIX &&
                strncmp(stAttr.tags, "addr=", strlen("addr=")) == 0) {
            status_t ret = mPolicyMixes.getInputMixForAttr(stAttr, &policyMix);
            if (ret != NO_ERROR) {
                return ret;
            }
            *inputType = API_INPUT_MIX_EXT_POLICY_REROUTE;
            device = AUDIO_DEVICE_IN_REMOTE_SUBMIX;
            address = String8(stAttr.tags + strlen("addr="));
        }
#else
    halInputSource = inputSource;

    // Explicit routing?
    sp<DeviceDescriptor> deviceDesc;
    for (size_t i = 0; i < mAvailableInputDevices.size(); i++) {
        if (mAvailableInputDevices[i]->getId() == selectedDeviceId) {
            deviceDesc = mAvailableInputDevices[i];
            break;
        }
    }
    mInputRoutes.addRoute(session, SessionRoute::STREAM_TYPE_NA, inputSource, deviceDesc, uid);

    if (inputSource == AUDIO_SOURCE_REMOTE_SUBMIX &&
            strncmp(attr->tags, "addr=", strlen("addr=")) == 0) {
        status_t ret = mPolicyMixes.getInputMixForAttr(*attr, &policyMix);
        if (ret != NO_ERROR) {
            return ret;
        }
        *inputType = API_INPUT_MIX_EXT_POLICY_REROUTE;
        device = AUDIO_DEVICE_IN_REMOTE_SUBMIX;
        address = String8(attr->tags + strlen("addr="));
    }
#endif
    else {
        device = getDeviceAndMixForInputSource(inputSource, &policyMix);
        if (device == AUDIO_DEVICE_NONE) {
            ALOGW("getInputForAttr() could not find device for source %d", inputSource);
            return BAD_VALUE;
        }
        if (policyMix != NULL) {
            address = policyMix->mDeviceAddress;
            if (policyMix->mMixType == MIX_TYPE_RECORDERS) {
                // there is an external policy, but this input is attached to a mix of recorders,
                // meaning it receives audio injected into the framework, so the recorder doesn't
                // know about it and is therefore considered "legacy"
                *inputType = API_INPUT_LEGACY;
            } else {
                // recording a mix of players defined by an external policy, we're rerouting for
                // an external policy
                *inputType = API_INPUT_MIX_EXT_POLICY_REROUTE;
            }
        } else if (audio_is_remote_submix_device(device)) {
            address = String8("0");
            *inputType = API_INPUT_MIX_CAPTURE;
        } else if (device == AUDIO_DEVICE_IN_TELEPHONY_RX) {
            *inputType = API_INPUT_TELEPHONY_RX;
        } else {
            *inputType = API_INPUT_LEGACY;
        }

    }

    *input = getInputForDevice(device, address, session, uid, inputSource,
                               samplingRate, format, channelMask, flags,
                               policyMix);
    if (*input == AUDIO_IO_HANDLE_NONE) {
        mInputRoutes.removeRoute(session);
        return INVALID_OPERATION;
    }
    ALOGV("getInputForAttr() returns input type = %d", *inputType);
    return NO_ERROR;
}


audio_io_handle_t AudioPolicyManager::getInputForDevice(audio_devices_t device,
                                                        String8 address,
                                                        audio_session_t session,
                                                        uid_t uid,
                                                        audio_source_t inputSource,
                                                        uint32_t samplingRate,
                                                        audio_format_t format,
                                                        audio_channel_mask_t channelMask,
                                                        audio_input_flags_t flags,
                                                        AudioMix *policyMix)
{
    audio_io_handle_t input = AUDIO_IO_HANDLE_NONE;
    audio_source_t halInputSource = inputSource;
    bool isSoundTrigger = false;

    if (inputSource == AUDIO_SOURCE_HOTWORD) {
        ssize_t index = mSoundTriggerSessions.indexOfKey(session);
        if (index >= 0) {
            input = mSoundTriggerSessions.valueFor(session);
            isSoundTrigger = true;
            flags = (audio_input_flags_t)(flags | AUDIO_INPUT_FLAG_HW_HOTWORD);
            ALOGV("SoundTrigger capture on session %d input %d", session, input);
        } else {
            halInputSource = AUDIO_SOURCE_VOICE_RECOGNITION;
        }
    }

    // find a compatible input profile (not necessarily identical in parameters)
    sp<IOProfile> profile;
    // samplingRate and flags may be updated by getInputProfile
    uint32_t profileSamplingRate = (samplingRate == 0) ? SAMPLE_RATE_HZ_DEFAULT : samplingRate;
    audio_format_t profileFormat = format;
    audio_channel_mask_t profileChannelMask = channelMask;
    audio_input_flags_t profileFlags = flags;
    for (;;) {
        profile = getInputProfile(device, address,
                                  profileSamplingRate, profileFormat, profileChannelMask,
                                  profileFlags);
        if (profile != 0) {
            break; // success
        } else if (profileFlags != AUDIO_INPUT_FLAG_NONE) {
            profileFlags = AUDIO_INPUT_FLAG_NONE; // retry
        } else { // fail
            ALOGW("getInputForDevice() could not find profile for device 0x%X,"
                  "samplingRate %u, format %#x, channelMask 0x%X, flags %#x",
                    device, samplingRate, format, channelMask, flags);
            return input;
        }
    }
    // Pick input sampling rate if not specified by client
    if (samplingRate == 0) {
        samplingRate = profileSamplingRate;
    }

    if (profile->getModuleHandle() == 0) {
        ALOGE("getInputForAttr(): HW module %s not opened", profile->getModuleName());
        return input;
    }

    sp<AudioSession> audioSession = new AudioSession(session,
                                                              inputSource,
                                                              format,
                                                              samplingRate,
                                                              channelMask,
                                                              flags,
                                                              uid,
                                                              isSoundTrigger,
                                                              policyMix, mpClientInterface);

// TODO enable input reuse
#if 0
    // reuse an open input if possible
    for (size_t i = 0; i < mInputs.size(); i++) {
        sp<AudioInputDescriptor> desc = mInputs.valueAt(i);
        // reuse input if it shares the same profile and same sound trigger attribute
        if (profile == desc->mProfile &&
            isSoundTrigger == desc->isSoundTrigger()) {

            sp<AudioSession> as = desc->getAudioSession(session);
            if (as != 0) {
                // do not allow unmatching properties on same session
                if (as->matches(audioSession)) {
                    as->changeOpenCount(1);
                } else {
                    ALOGW("getInputForDevice() record with different attributes"
                          " exists for session %d", session);
                    return input;
                }
            } else {
                desc->addAudioSession(session, audioSession);
            }
            ALOGV("getInputForDevice() reusing input %d", mInputs.keyAt(i));
            return mInputs.keyAt(i);
        }
    }
#endif

    audio_config_t config = AUDIO_CONFIG_INITIALIZER;
    config.sample_rate = profileSamplingRate;
    config.channel_mask = profileChannelMask;
    config.format = profileFormat;

    status_t status = mpClientInterface->openInput(profile->getModuleHandle(),
                                                   &input,
                                                   &config,
                                                   &device,
                                                   address,
                                                   halInputSource,
                                                   profileFlags);

    // only accept input with the exact requested set of parameters
    if (status != NO_ERROR || input == AUDIO_IO_HANDLE_NONE ||
        (profileSamplingRate != config.sample_rate) ||
        !audio_formats_match(profileFormat, config.format) ||
        (profileChannelMask != config.channel_mask)) {
        ALOGW("getInputForAttr() failed opening input: samplingRate %d"
              ", format %d, channelMask %x",
                samplingRate, format, channelMask);
        if (input != AUDIO_IO_HANDLE_NONE) {
            mpClientInterface->closeInput(input);
        }
        return AUDIO_IO_HANDLE_NONE;
    }

    sp<AudioInputDescriptor> inputDesc = new AudioInputDescriptor(profile);
    inputDesc->mSamplingRate = profileSamplingRate;
    inputDesc->mFormat = profileFormat;
    inputDesc->mChannelMask = profileChannelMask;
    inputDesc->mDevice = device;
    inputDesc->mPolicyMix = policyMix;
    inputDesc->addAudioSession(session, audioSession);

    addInput(input, inputDesc);
    mpClientInterface->onAudioPortListUpdate();

    return input;
}

status_t AudioPolicyManager::startInput(audio_io_handle_t input,
                                        audio_session_t session)
{
    ALOGV("startInput() input %d", input);
    ssize_t index = mInputs.indexOfKey(input);
    if (index < 0) {
        ALOGW("startInput() unknown input %d", input);
        return BAD_VALUE;
    }
    sp<AudioInputDescriptor> inputDesc = mInputs.valueAt(index);

    sp<AudioSession> audioSession = inputDesc->getAudioSession(session);
    if (audioSession == 0) {
        ALOGW("startInput() unknown session %d on input %d", session, input);
        return BAD_VALUE;
    }

    // virtual input devices are compatible with other input devices
    if (!is_virtual_input_device(inputDesc->mDevice)) {

        // for a non-virtual input device, check if there is another (non-virtual) active input
        audio_io_handle_t activeInput = mInputs.getActiveInput();
        if (activeInput != 0 && activeInput != input) {

            // If the already active input uses AUDIO_SOURCE_HOTWORD then it is closed,
            // otherwise the active input continues and the new input cannot be started.
            sp<AudioInputDescriptor> activeDesc = mInputs.valueFor(activeInput);
            if ((activeDesc->inputSource() == AUDIO_SOURCE_HOTWORD) &&
                    !activeDesc->hasPreemptedSession(session)) {
                ALOGW("startInput(%d) preempting low-priority input %d", input, activeInput);
                //FIXME: consider all active sessions
                AudioSessionCollection activeSessions = activeDesc->getActiveAudioSessions();
                audio_session_t activeSession = activeSessions.keyAt(0);
                SortedVector<audio_session_t> sessions =
                                           activeDesc->getPreemptedSessions();
                sessions.add(activeSession);
                inputDesc->setPreemptedSessions(sessions);
                stopInput(activeInput, activeSession);
                releaseInput(activeInput, activeSession);
            } else {
#ifdef MTK_AUDIO
                // Support multiple streamIn, however there is a GTS item to verify preemptive property for HOTWORD
                if (inputDesc->inputSource() == AUDIO_SOURCE_HOTWORD) {
                    ALOGW("startInput(AUDIO_SOURCE_HOTWORD) invalid, already started high-priority input %d", activeInput);
                    return INVALID_OPERATION;
                }
#else
                ALOGE("startInput(%d) failed: other input %d already started", input, activeInput);
                return INVALID_OPERATION;
#endif
            }
        }

        // Do not allow capture if an active voice call is using a software patch and
        // the call TX source device is on the same HW module.
        // FIXME: would be better to refine to only inputs whose profile connects to the
        // call TX device but this information is not in the audio patch
        if (mCallTxPatch != 0 &&
            inputDesc->getModuleHandle() == mCallTxPatch->mPatch.sources[0].ext.device.hw_module) {
            return INVALID_OPERATION;
        }
    }

    // Routing?
    mInputRoutes.incRouteActivity(session);

    if (!inputDesc->isActive() || mInputRoutes.hasRouteChanged(session)) {
        // if input maps to a dynamic policy with an activity listener, notify of state change
        if ((inputDesc->mPolicyMix != NULL)
                && ((inputDesc->mPolicyMix->mCbFlags & AudioMix::kCbFlagNotifyActivity) != 0)) {
            mpClientInterface->onDynamicPolicyMixStateUpdate(inputDesc->mPolicyMix->mDeviceAddress,
                    MIX_STATE_MIXING);
        }

        if (mInputs.activeInputsCount() == 0) {
            SoundTrigger::setCaptureState(true);
        }
        setInputDevice(input, getNewInputDevice(input), true /* force */);

        // automatically enable the remote submix output when input is started if not
        // used by a policy mix of type MIX_TYPE_RECORDERS
        // For remote submix (a virtual device), we open only one input per capture request.
        if (audio_is_remote_submix_device(inputDesc->mDevice)) {
            String8 address = String8("");
            if (inputDesc->mPolicyMix == NULL) {
                address = String8("0");
            } else if (inputDesc->mPolicyMix->mMixType == MIX_TYPE_PLAYERS) {
                address = inputDesc->mPolicyMix->mDeviceAddress;
            }
            if (address != "") {
                setDeviceConnectionStateInt(AUDIO_DEVICE_OUT_REMOTE_SUBMIX,
                        AUDIO_POLICY_DEVICE_STATE_AVAILABLE,
                        address, "remote-submix");
            }
#ifdef MTK_CROSSMOUNT_SUPPORT
            // If sink of Mic starts to record, we notify CrossMount begin to write data to audio track
            if (((address == String8("CrossMount-Mic")) && mAudioPolicyVendorControl.getCrossMountMicAudioMixerEnable())) {
                AudioParameter param;
                ALOGD("InvalidateCrossMountTrack");
                param.addInt(String8("InvalidateCrossMountTrack"), 1);
                mpClientInterface->setParameters (0 , param.toString(), 0);
                ALOGD("CrossMountMIC-Enable Record");
                CrossMountSink::notifyMicStatus(true);
            }
#endif

        }
    }

    ALOGV("AudioPolicyManager::startInput() input source = %d", audioSession->inputSource());

    audioSession->changeActiveCount(1);
    return NO_ERROR;
}

status_t AudioPolicyManager::stopInput(audio_io_handle_t input,
                                       audio_session_t session)
{
    ALOGV("stopInput() input %d", input);
    ssize_t index = mInputs.indexOfKey(input);
    if (index < 0) {
        ALOGW("stopInput() unknown input %d", input);
        return BAD_VALUE;
    }
    sp<AudioInputDescriptor> inputDesc = mInputs.valueAt(index);

    sp<AudioSession> audioSession = inputDesc->getAudioSession(session);
    if (index < 0) {
        ALOGW("stopInput() unknown session %d on input %d", session, input);
        return BAD_VALUE;
    }

    if (audioSession->activeCount() == 0) {
        ALOGW("stopInput() input %d already stopped", input);
        return INVALID_OPERATION;
    }

    audioSession->changeActiveCount(-1);

    // Routing?
    mInputRoutes.decRouteActivity(session);

    if (!inputDesc->isActive()) {
        // if input maps to a dynamic policy with an activity listener, notify of state change
        if ((inputDesc->mPolicyMix != NULL)
                && ((inputDesc->mPolicyMix->mCbFlags & AudioMix::kCbFlagNotifyActivity) != 0)) {
            mpClientInterface->onDynamicPolicyMixStateUpdate(inputDesc->mPolicyMix->mDeviceAddress,
                    MIX_STATE_IDLE);
        }

        // automatically disable the remote submix output when input is stopped if not
        // used by a policy mix of type MIX_TYPE_RECORDERS
        if (audio_is_remote_submix_device(inputDesc->mDevice)) {
            String8 address = String8("");
            if (inputDesc->mPolicyMix == NULL) {
                address = String8("0");
            } else if (inputDesc->mPolicyMix->mMixType == MIX_TYPE_PLAYERS) {
                address = inputDesc->mPolicyMix->mDeviceAddress;
            }
            if (address != "") {
#ifdef MTK_CROSSMOUNT_SUPPORT   // If sink of Mic stop recording, we notify CrossMount stop writing data to audio track
                if (mAudioPolicyVendorControl.getCrossMountMicLocalPlayback() == false &&
                    (((address == String8("CrossMount-Mic")) && mAudioPolicyVendorControl.getCrossMountMicAudioMixerEnable()))) {
                    ALOGD("CrossMountMIC-Disable Record");
                    CrossMountSink::notifyMicStatus(false);
                }
                bool skipdisconnectoutput = false;
                if (inputDesc->mPolicyMix == NULL) {
                    for(size_t input_index = 0; input_index < mInputs.size(); input_index++) {
                        sp<AudioInputDescriptor> otherinputDesc = mInputs.valueAt(input_index);
                        if (audio_is_remote_submix_device(otherinputDesc->mDevice) && otherinputDesc->isActive() && otherinputDesc->mPolicyMix == NULL) {
                            skipdisconnectoutput = true;
                            break;
                        }
                    }
                }
                if (skipdisconnectoutput == false) {
                    setDeviceConnectionStateInt(AUDIO_DEVICE_OUT_REMOTE_SUBMIX,
                                         AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE,
                                         address, "remote-submix");
                }
#else
                setDeviceConnectionStateInt(AUDIO_DEVICE_OUT_REMOTE_SUBMIX,
                                         AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE,
                                         address, "remote-submix");
#endif
            }
        }

        resetInputDevice(input);

        if (mInputs.activeInputsCount() == 0) {
            SoundTrigger::setCaptureState(false);
        }
        inputDesc->clearPreemptedSessions();
    }
    return NO_ERROR;
}

void AudioPolicyManager::releaseInput(audio_io_handle_t input,
                                      audio_session_t session)
{

    ALOGV("releaseInput() %d", input);
    ssize_t index = mInputs.indexOfKey(input);
    if (index < 0) {
        ALOGW("releaseInput() releasing unknown input %d", input);
        return;
    }

    // Routing
    mInputRoutes.removeRoute(session);

    sp<AudioInputDescriptor> inputDesc = mInputs.valueAt(index);
    ALOG_ASSERT(inputDesc != 0);

    sp<AudioSession> audioSession = inputDesc->getAudioSession(session);
    if (index < 0) {
        ALOGW("releaseInput() unknown session %d on input %d", session, input);
        return;
    }

    if (audioSession->openCount() == 0) {
        ALOGW("releaseInput() invalid open count %d on session %d",
              audioSession->openCount(), session);
        return;
    }

    if (audioSession->changeOpenCount(-1) == 0) {
        inputDesc->removeAudioSession(session);
    }

    if (inputDesc->getOpenRefCount() > 0) {
        ALOGV("releaseInput() exit > 0");
        return;
    }

    closeInput(input);
    mpClientInterface->onAudioPortListUpdate();
    ALOGV("releaseInput() exit");
}

void AudioPolicyManager::closeAllInputs() {
    bool patchRemoved = false;

    for(size_t input_index = 0; input_index < mInputs.size(); input_index++) {
        sp<AudioInputDescriptor> inputDesc = mInputs.valueAt(input_index);
        ssize_t patch_index = mAudioPatches.indexOfKey(inputDesc->getPatchHandle());
        if (patch_index >= 0) {
            sp<AudioPatch> patchDesc = mAudioPatches.valueAt(patch_index);
            status_t status = mpClientInterface->releaseAudioPatch(patchDesc->mAfPatchHandle, 0);
            mAudioPatches.removeItemsAt(patch_index);
            patchRemoved = true;
        }
#ifdef MTK_AUDIO
        ALOGD("closeAllInputs(): %d",mInputs.keyAt(input_index));
        // when remote submix input close, it should close its output.
        // Or stopinput won't close its output because input handle is invalid
        // automatically disable the remote submix output when input is stopped if not
        // used by a policy mix of type MIX_TYPE_RECORDERS
        if (audio_is_remote_submix_device(inputDesc->mDevice)) {
            String8 address = String8("");
            if (inputDesc->mPolicyMix == NULL) {
                address = String8("0");
            } else if (inputDesc->mPolicyMix->mMixType == MIX_TYPE_PLAYERS) {
                address = inputDesc->mPolicyMix->mDeviceAddress;
            }
            if (address != "") {
#ifdef MTK_CROSSMOUNT_SUPPORT
                if (mAudioPolicyVendorControl.getCrossMountMicLocalPlayback() == false &&
                    (((address == String8("CrossMount-Mic")) && mAudioPolicyVendorControl.getCrossMountMicAudioMixerEnable()))) {
                    ALOGD("CrossMountMIC-Disable Record");
                    CrossMountSink::notifyMicStatus(false);
                }
#endif
                setDeviceConnectionStateInt(AUDIO_DEVICE_OUT_REMOTE_SUBMIX,
                                         AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE,
                                         address, "remote-submix");
            }
            ALOGD("closeAllInputs() for audio_is_remote_submix_device done: %d",mInputs.keyAt(input_index));
        }
#endif
        mpClientInterface->closeInput(mInputs.keyAt(input_index));
    }
    mInputs.clear();
    SoundTrigger::setCaptureState(false);
    nextAudioPortGeneration();

    if (patchRemoved) {
        mpClientInterface->onAudioPatchListUpdate();
    }
}

void AudioPolicyManager::initStreamVolume(audio_stream_type_t stream,
                                            int indexMin,
                                            int indexMax)
{
    ALOGV("initStreamVolume() stream %d, min %d, max %d", stream , indexMin, indexMax);
#if defined(MTK_AUDIO) && !defined(MTK_NEW_VOL_CONTROL)  //  Remapping M min index = 1 to MTK min index = 0
    if (stream == AUDIO_STREAM_VOICE_CALL) {
        if (indexMin > 0) {
            mNeedRemapVoiceVolumeIndex = true;
            indexMin--;
            indexMax--;
            ALOGV("Correct stream %d, min %d, max %d", stream , indexMin, indexMax);
        }
    }
#endif
    mVolumeCurves->initStreamVolume(stream, indexMin, indexMax);

    // initialize other private stream volumes which follow this one
    for (int curStream = 0; curStream < AUDIO_STREAM_FOR_POLICY_CNT; curStream++) {
        if (!streamsMatchForvolume(stream, (audio_stream_type_t)curStream)) {
            continue;
        }
        mVolumeCurves->initStreamVolume((audio_stream_type_t)curStream, indexMin, indexMax);
    }
}

status_t AudioPolicyManager::setStreamVolumeIndex(audio_stream_type_t stream,
                                                  int index,
                                                  audio_devices_t device)
{

#if defined(MTK_AUDIO)
#ifndef MTK_NEW_VOL_CONTROL  //  Remapping M min index = 1 to MTK min index = 0
    if (stream == AUDIO_STREAM_VOICE_CALL) {
        if (mNeedRemapVoiceVolumeIndex == true) {
            index--;
            // ALOGV("Correct stream = %d index = %d device = 0x%x",stream,index,device);
        }
    }
#endif
#endif
    if ((index < mVolumeCurves->getVolumeIndexMin(stream)) ||
            (index > mVolumeCurves->getVolumeIndexMax(stream))) {
#if defined(MTK_AUDIO)
        ALOGE("setStreamVolumeIndex stream = %d index = %d device = 0x%x Min %d Max %d Err", stream, index, device, mVolumeCurves->getVolumeIndexMin(stream), mVolumeCurves->getVolumeIndexMax(stream));
#endif
        return BAD_VALUE;
    }
    if (!audio_is_output_device(device)) {
        return BAD_VALUE;
#if defined(MTK_AUDIO)
        ALOGE("setStreamVolumeIndex stream = %d index = %d device = 0x%x Err",stream,index,device);
#endif
    }

    // Force max volume if stream cannot be muted
    if (!mVolumeCurves->canBeMuted(stream)) index = mVolumeCurves->getVolumeIndexMax(stream);

    ALOGVV("setStreamVolumeIndex() stream %d, device %08x, index %d",
          stream, device, index);

    // update other private stream volumes which follow this one
    for (int curStream = 0; curStream < AUDIO_STREAM_FOR_POLICY_CNT; curStream++) {
        if (!streamsMatchForvolume(stream, (audio_stream_type_t)curStream)) {
            continue;
        }
        mVolumeCurves->addCurrentVolumeIndex((audio_stream_type_t)curStream, device, index);
    }

    // update volume on all outputs and streams matching the following:
    // - The requested stream (or a stream matching for volume control) is active on the output
    // - The device (or devices) selected by the strategy corresponding to this stream includes
    // the requested device
    // - For non default requested device, currently selected device on the output is either the
    // requested device or one of the devices selected by the strategy
    // - For default requested device (AUDIO_DEVICE_OUT_DEFAULT_FOR_VOLUME), apply volume only if
    // no specific device volume value exists for currently selected device.
    status_t status = NO_ERROR;
    for (size_t i = 0; i < mOutputs.size(); i++) {
        sp<SwAudioOutputDescriptor> desc = mOutputs.valueAt(i);
        audio_devices_t curDevice = Volume::getDeviceForVolume(desc->device());
#ifdef MTK_AUDIO
        if (desc == mPrimaryOutput) {
            if ((desc->device() == (AUDIO_DEVICE_OUT_SPEAKER|AUDIO_DEVICE_OUT_WIRED_HEADSET))
                || (desc->device() == (AUDIO_DEVICE_OUT_SPEAKER|AUDIO_DEVICE_OUT_WIRED_HEADPHONE))) {
                    curDevice = desc->device();
                }
        }
#endif
        for (int curStream = 0; curStream < AUDIO_STREAM_FOR_POLICY_CNT; curStream++) {
            if (!streamsMatchForvolume(stream, (audio_stream_type_t)curStream)) {
                continue;
            }
            if (!(desc->isStreamActive((audio_stream_type_t)curStream) ||
                    (isInCall() && (curStream == AUDIO_STREAM_VOICE_CALL)))) {
                continue;
            }
            routing_strategy curStrategy = getStrategy((audio_stream_type_t)curStream);
            audio_devices_t curStreamDevice = getDeviceForStrategy(curStrategy, true /*fromCache*/);
            if ((curStreamDevice & device) == 0) {
#ifdef MTK_AUDIO  // Google Issue. side effect of M->N, ALPS02874222, it should sync with ALPS02902310
                if (device != AUDIO_DEVICE_OUT_DEFAULT_FOR_VOLUME) {
                    continue;
                }
#else
                continue;
#endif
            }
            bool applyDefault = false;
            if (device != AUDIO_DEVICE_OUT_DEFAULT_FOR_VOLUME) {
                curStreamDevice |= device;
            } else if (!mVolumeCurves->hasVolumeIndexForDevice(
                    stream, Volume::getDeviceForVolume(curStreamDevice))) {
                applyDefault = true;
            }

            if (applyDefault || ((curDevice & curStreamDevice) != 0)) {
                status_t volStatus =
                        checkAndSetVolume((audio_stream_type_t)curStream, index, desc, curDevice);
                if (volStatus != NO_ERROR) {
                    status = volStatus;
                }
            }
        }
    }
    return status;
}

status_t AudioPolicyManager::getStreamVolumeIndex(audio_stream_type_t stream,
                                                      int *index,
                                                      audio_devices_t device)
{
    if (index == NULL) {
        return BAD_VALUE;
    }
    if (!audio_is_output_device(device)) {
        return BAD_VALUE;
    }
    // if device is AUDIO_DEVICE_OUT_DEFAULT_FOR_VOLUME, return volume for device corresponding to
    // the strategy the stream belongs to.
    if (device == AUDIO_DEVICE_OUT_DEFAULT_FOR_VOLUME) {
        device = getDeviceForStrategy(getStrategy(stream), true /*fromCache*/);
    }
    device = Volume::getDeviceForVolume(device);

    *index =  mVolumeCurves->getVolumeIndex(stream, device);
    ALOGV("getStreamVolumeIndex() stream %d device %08x index %d", stream, device, *index);
#if defined(MTK_AUDIO) && !defined(MTK_NEW_VOL_CONTROL)  //  Remapping M min index = 1 to MTK min index = 0
    if (stream == AUDIO_STREAM_VOICE_CALL) {
        if (mNeedRemapVoiceVolumeIndex == true) {
            *index = *index + 1;
            ALOGV("Correct stream %d device %08x index %d", stream, device, *index);
        }
    }
#endif
    return NO_ERROR;
}

audio_io_handle_t AudioPolicyManager::selectOutputForEffects(
                                            const SortedVector<audio_io_handle_t>& outputs)
{
    // select one output among several suitable for global effects.
    // The priority is as follows:
    // 1: An offloaded output. If the effect ends up not being offloadable,
    //    AudioFlinger will invalidate the track and the offloaded output
    //    will be closed causing the effect to be moved to a PCM output.
    // 2: A deep buffer output
    // 3: the first output in the list

    if (outputs.size() == 0) {
        return 0;
    }

    audio_io_handle_t outputOffloaded = 0;
    audio_io_handle_t outputDeepBuffer = 0;

    for (size_t i = 0; i < outputs.size(); i++) {
        sp<SwAudioOutputDescriptor> desc = mOutputs.valueFor(outputs[i]);
        ALOGV("selectOutputForEffects outputs[%zu] flags %x", i, desc->mFlags);
        if ((desc->mFlags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) != 0) {
            outputOffloaded = outputs[i];
        }
        if ((desc->mFlags & AUDIO_OUTPUT_FLAG_DEEP_BUFFER) != 0) {
            outputDeepBuffer = outputs[i];
        }
    }

    ALOGV("selectOutputForEffects outputOffloaded %d outputDeepBuffer %d",
          outputOffloaded, outputDeepBuffer);
    if (outputOffloaded != 0) {
        return outputOffloaded;
    }
    if (outputDeepBuffer != 0) {
        return outputDeepBuffer;
    }

    return outputs[0];
}

audio_io_handle_t AudioPolicyManager::getOutputForEffect(const effect_descriptor_t *desc)
{
    // apply simple rule where global effects are attached to the same output as MUSIC streams

    routing_strategy strategy = getStrategy(AUDIO_STREAM_MUSIC);
    audio_devices_t device = getDeviceForStrategy(strategy, false /*fromCache*/);
    SortedVector<audio_io_handle_t> dstOutputs = getOutputsForDevice(device, mOutputs);

    audio_io_handle_t output = selectOutputForEffects(dstOutputs);
    ALOGV("getOutputForEffect() got output %d for fx %s flags %x",
          output, (desc == NULL) ? "unspecified" : desc->name,  (desc == NULL) ? 0 : desc->flags);

    return output;
}

status_t AudioPolicyManager::registerEffect(const effect_descriptor_t *desc,
                                audio_io_handle_t io,
                                uint32_t strategy,
                                int session,
                                int id)
{
    ssize_t index = mOutputs.indexOfKey(io);
    if (index < 0) {
        index = mInputs.indexOfKey(io);
        if (index < 0) {
            ALOGW("registerEffect() unknown io %d", io);
            return INVALID_OPERATION;
        }
    }
    return mEffects.registerEffect(desc, io, strategy, session, id);
}

bool AudioPolicyManager::isStreamActive(audio_stream_type_t stream, uint32_t inPastMs) const
{
    bool active = false;
    for (int curStream = 0; curStream < AUDIO_STREAM_FOR_POLICY_CNT && !active; curStream++) {
        if (!streamsMatchForvolume(stream, (audio_stream_type_t)curStream)) {
            continue;
        }
        active = mOutputs.isStreamActive((audio_stream_type_t)curStream, inPastMs);
    }
    return active;
}

bool AudioPolicyManager::isStreamActiveRemotely(audio_stream_type_t stream, uint32_t inPastMs) const
{
    return mOutputs.isStreamActiveRemotely(stream, inPastMs);
}

bool AudioPolicyManager::isSourceActive(audio_source_t source) const
{
    for (size_t i = 0; i < mInputs.size(); i++) {
        const sp<AudioInputDescriptor>  inputDescriptor = mInputs.valueAt(i);
        if (inputDescriptor->isSourceActive(source)) {
            return true;
        }
    }
    return false;
}

// Register a list of custom mixes with their attributes and format.
// When a mix is registered, corresponding input and output profiles are
// added to the remote submix hw module. The profile contains only the
// parameters (sampling rate, format...) specified by the mix.
// The corresponding input remote submix device is also connected.
//
// When a remote submix device is connected, the address is checked to select the
// appropriate profile and the corresponding input or output stream is opened.
//
// When capture starts, getInputForAttr() will:
//  - 1 look for a mix matching the address passed in attribtutes tags if any
//  - 2 if none found, getDeviceForInputSource() will:
//     - 2.1 look for a mix matching the attributes source
//     - 2.2 if none found, default to device selection by policy rules
// At this time, the corresponding output remote submix device is also connected
// and active playback use cases can be transferred to this mix if needed when reconnecting
// after AudioTracks are invalidated
//
// When playback starts, getOutputForAttr() will:
//  - 1 look for a mix matching the address passed in attribtutes tags if any
//  - 2 if none found, look for a mix matching the attributes usage
//  - 3 if none found, default to device and output selection by policy rules.

status_t AudioPolicyManager::registerPolicyMixes(Vector<AudioMix> mixes)
{
    ALOGV("registerPolicyMixes() %zu mix(es)", mixes.size());
    status_t res = NO_ERROR;

    sp<HwModule> rSubmixModule;
    // examine each mix's route type
    for (size_t i = 0; i < mixes.size(); i++) {
        // we only support MIX_ROUTE_FLAG_LOOP_BACK or MIX_ROUTE_FLAG_RENDER, not the combination
        if ((mixes[i].mRouteFlags & MIX_ROUTE_FLAG_ALL) == MIX_ROUTE_FLAG_ALL) {
            res = INVALID_OPERATION;
            break;
        }
        if ((mixes[i].mRouteFlags & MIX_ROUTE_FLAG_LOOP_BACK) == MIX_ROUTE_FLAG_LOOP_BACK) {
            // Loop back through "remote submix"
            if (rSubmixModule == 0) {
                for (size_t j = 0; i < mHwModules.size(); j++) {
                    if (strcmp(AUDIO_HARDWARE_MODULE_ID_REMOTE_SUBMIX, mHwModules[j]->mName) == 0
                            && mHwModules[j]->mHandle != 0) {
                        rSubmixModule = mHwModules[j];
                        break;
                    }
                }
            }

            ALOGV("registerPolicyMixes() mix %zu of %zu is LOOP_BACK", i, mixes.size());

            if (rSubmixModule == 0) {
                ALOGE(" Unable to find audio module for submix, aborting mix %zu registration", i);
                res = INVALID_OPERATION;
                break;
            }

            String8 address = mixes[i].mDeviceAddress;

            if (mPolicyMixes.registerMix(address, mixes[i], 0 /*output desc*/) != NO_ERROR) {
                ALOGE(" Error registering mix %zu for address %s", i, address.string());
                res = INVALID_OPERATION;
                break;
            }
#ifdef MTK_CROSSMOUNT_SUPPORT   //  CrossMount init Mic crossmount processing
        if (address == String8("CrossMount-Mic")) {
            ALOGD("CrossMount-Mic Mixer get");
            mAudioPolicyVendorControl.setCrossMountMicAudioMixerEnable(true);
            ALOGD("mMixType %d, should be 0", mixes[i].mMixType);//MIX_TYPE_PLAYERS
            ALOGD("mRouteFlags %d, should be 1", mixes[i].mRouteFlags);//ROUTE_FLAG_RENDER
            ALOGD("mFormat.sample_rate %d, should be 48000", mixes[i].mFormat.sample_rate);//48000
            ALOGD("mFormat.channel_mask 0x%x, should be 0x0c", mixes[i].mFormat.channel_mask);//AUDIO_CHANNEL_IN_STEREO
            ALOGD("mFormat.format 0x%x should be 0x01", mixes[i].mFormat.format);//AUDIO_FORMAT_PCM_16_BIT
            ALOGD("mFormat.frame_count %zu, should be 0", mixes[i].mFormat.frame_count);//0
        } else {
            ALOGD("CrossMount-Mic Mixer miss");
        }
#endif
            audio_config_t outputConfig = mixes[i].mFormat;
            audio_config_t inputConfig = mixes[i].mFormat;
            // NOTE: audio flinger mixer does not support mono output: configure remote submix HAL in
            // stereo and let audio flinger do the channel conversion if needed.
            outputConfig.channel_mask = AUDIO_CHANNEL_OUT_STEREO;
            inputConfig.channel_mask = AUDIO_CHANNEL_IN_STEREO;
            rSubmixModule->addOutputProfile(address, &outputConfig,
                    AUDIO_DEVICE_OUT_REMOTE_SUBMIX, address);
            rSubmixModule->addInputProfile(address, &inputConfig,
                    AUDIO_DEVICE_IN_REMOTE_SUBMIX, address);

            if (mixes[i].mMixType == MIX_TYPE_PLAYERS) {
                setDeviceConnectionStateInt(AUDIO_DEVICE_IN_REMOTE_SUBMIX,
                        AUDIO_POLICY_DEVICE_STATE_AVAILABLE,
                        address.string(), "remote-submix");
            } else {
                setDeviceConnectionStateInt(AUDIO_DEVICE_OUT_REMOTE_SUBMIX,
                        AUDIO_POLICY_DEVICE_STATE_AVAILABLE,
                        address.string(), "remote-submix");
            }
        } else if ((mixes[i].mRouteFlags & MIX_ROUTE_FLAG_RENDER) == MIX_ROUTE_FLAG_RENDER) {
            String8 address = mixes[i].mDeviceAddress;
            audio_devices_t device = mixes[i].mDeviceType;
            ALOGV(" registerPolicyMixes() mix %zu of %zu is RENDER, dev=0x%X addr=%s",
                    i, mixes.size(), device, address.string());

            bool foundOutput = false;
            for (size_t j = 0 ; j < mOutputs.size() ; j++) {
                sp<SwAudioOutputDescriptor> desc = mOutputs.valueAt(j);
                sp<AudioPatch> patch = mAudioPatches.valueFor(desc->getPatchHandle());
                if ((patch != 0) && (patch->mPatch.num_sinks != 0)
                        && (patch->mPatch.sinks[0].type == AUDIO_PORT_TYPE_DEVICE)
                        && (patch->mPatch.sinks[0].ext.device.type == device)
                        && (strncmp(patch->mPatch.sinks[0].ext.device.address, address.string(),
                                AUDIO_DEVICE_MAX_ADDRESS_LEN) == 0)) {
                    if (mPolicyMixes.registerMix(address, mixes[i], desc) != NO_ERROR) {
                        res = INVALID_OPERATION;
                    } else {
                        foundOutput = true;
                    }
                    break;
                }
            }

            if (res != NO_ERROR) {
                ALOGE(" Error registering mix %zu for device 0x%X addr %s",
                        i, device, address.string());
                res = INVALID_OPERATION;
                break;
            } else if (!foundOutput) {
                ALOGE(" Output not found for mix %zu for device 0x%X addr %s",
                        i, device, address.string());
                res = INVALID_OPERATION;
                break;
            }
        }
    }
    if (res != NO_ERROR) {
        unregisterPolicyMixes(mixes);
    }
    return res;
}

status_t AudioPolicyManager::unregisterPolicyMixes(Vector<AudioMix> mixes)
{
    ALOGV("unregisterPolicyMixes() num mixes %zu", mixes.size());
    status_t res = NO_ERROR;
    sp<HwModule> rSubmixModule;
    // examine each mix's route type
    for (size_t i = 0; i < mixes.size(); i++) {
        if ((mixes[i].mRouteFlags & MIX_ROUTE_FLAG_LOOP_BACK) == MIX_ROUTE_FLAG_LOOP_BACK) {

            if (rSubmixModule == 0) {
                for (size_t j = 0; i < mHwModules.size(); j++) {
                    if (strcmp(AUDIO_HARDWARE_MODULE_ID_REMOTE_SUBMIX, mHwModules[j]->mName) == 0
                            && mHwModules[j]->mHandle != 0) {
                        rSubmixModule = mHwModules[j];
                        break;
                    }
                }
            }
            if (rSubmixModule == 0) {
                res = INVALID_OPERATION;
                continue;
            }

            String8 address = mixes[i].mDeviceAddress;

            if (mPolicyMixes.unregisterMix(address) != NO_ERROR) {
                res = INVALID_OPERATION;
                continue;
            }

#ifdef MTK_CROSSMOUNT_SUPPORT   //  CrossMount deinit Mic crossmount processing
        if (address == String8("CrossMount-Mic") && mAudioPolicyVendorControl.getCrossMountMicAudioMixerEnable()) {
            ALOGD("CrossMount-Mic Mixer dismiss");
            mAudioPolicyVendorControl.setCrossMountMicAudioMixerEnable(false);
        }
#endif
            if (getDeviceConnectionState(AUDIO_DEVICE_IN_REMOTE_SUBMIX, address.string()) ==
                    AUDIO_POLICY_DEVICE_STATE_AVAILABLE)  {
                setDeviceConnectionStateInt(AUDIO_DEVICE_IN_REMOTE_SUBMIX,
                        AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE,
                        address.string(), "remote-submix");
            }
            if (getDeviceConnectionState(AUDIO_DEVICE_OUT_REMOTE_SUBMIX, address.string()) ==
                    AUDIO_POLICY_DEVICE_STATE_AVAILABLE)  {
                setDeviceConnectionStateInt(AUDIO_DEVICE_OUT_REMOTE_SUBMIX,
                        AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE,
                        address.string(), "remote-submix");
            }
            rSubmixModule->removeOutputProfile(address);
            rSubmixModule->removeInputProfile(address);

        } if ((mixes[i].mRouteFlags & MIX_ROUTE_FLAG_RENDER) == MIX_ROUTE_FLAG_RENDER) {
            if (mPolicyMixes.unregisterMix(mixes[i].mDeviceAddress) != NO_ERROR) {
                res = INVALID_OPERATION;
                continue;
            }
        }
    }
    return res;
}


status_t AudioPolicyManager::dump(int fd)
{
    const size_t SIZE = 256;
    char buffer[SIZE];
    String8 result;

    snprintf(buffer, SIZE, "\nAudioPolicyManager Dump: %p\n", this);
    result.append(buffer);

    snprintf(buffer, SIZE, " Primary Output: %d\n",
             hasPrimaryOutput() ? mPrimaryOutput->mIoHandle : AUDIO_IO_HANDLE_NONE);
    result.append(buffer);
    snprintf(buffer, SIZE, " Phone state: %d\n", mEngine->getPhoneState());
    result.append(buffer);
    snprintf(buffer, SIZE, " Force use for communications %d\n",
             mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_COMMUNICATION));
    result.append(buffer);
    snprintf(buffer, SIZE, " Force use for media %d\n", mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_MEDIA));
    result.append(buffer);
    snprintf(buffer, SIZE, " Force use for record %d\n", mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_RECORD));
    result.append(buffer);
    snprintf(buffer, SIZE, " Force use for dock %d\n", mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_DOCK));
    result.append(buffer);
    snprintf(buffer, SIZE, " Force use for system %d\n", mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_SYSTEM));
    result.append(buffer);
    snprintf(buffer, SIZE, " Force use for hdmi system audio %d\n",
            mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_HDMI_SYSTEM_AUDIO));
    result.append(buffer);
    snprintf(buffer, SIZE, " Force use for encoded surround output %d\n",
            mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_ENCODED_SURROUND));
    result.append(buffer);
    snprintf(buffer, SIZE, " TTS output %s\n", mTtsOutputAvailable ? "available" : "not available");
    result.append(buffer);
    snprintf(buffer, SIZE, " Master mono: %s\n", mMasterMono ? "on" : "off");
    result.append(buffer);

    write(fd, result.string(), result.size());

    mAvailableOutputDevices.dump(fd, String8("Available output"));
    mAvailableInputDevices.dump(fd, String8("Available input"));
    mHwModules.dump(fd);
    mOutputs.dump(fd);
    mInputs.dump(fd);
    mVolumeCurves->dump(fd);
    mEffects.dump(fd);
    mAudioPatches.dump(fd);

    return NO_ERROR;
}

// This function checks for the parameters which can be offloaded.
// This can be enhanced depending on the capability of the DSP and policy
// of the system.
bool AudioPolicyManager::isOffloadSupported(const audio_offload_info_t& offloadInfo)
{
    ALOGV("isOffloadSupported: SR=%u, CM=0x%x, Format=0x%x, StreamType=%d,"
     " BitRate=%u, duration=%" PRId64 " us, has_video=%d",
     offloadInfo.sample_rate, offloadInfo.channel_mask,
     offloadInfo.format,
     offloadInfo.stream_type, offloadInfo.bit_rate, offloadInfo.duration_us,
     offloadInfo.has_video);

    if (mMasterMono) {
        return false; // no offloading if mono is set.
    }

    // Check if offload has been disabled
    char propValue[PROPERTY_VALUE_MAX];
    if (property_get("audio.offload.disable", propValue, "0")) {
        if (atoi(propValue) != 0) {
            ALOGV("offload disabled by audio.offload.disable=%s", propValue );
            return false;
        }
    }

    // Check if stream type is music, then only allow offload as of now.
    if (offloadInfo.stream_type != AUDIO_STREAM_MUSIC)
    {
        ALOGV("isOffloadSupported: stream_type != MUSIC, returning false");
        return false;
    }

    //TODO: enable audio offloading with video when ready
    const bool allowOffloadWithVideo =
            property_get_bool("audio.offload.video", false /* default_value */);
    if (offloadInfo.has_video && !allowOffloadWithVideo) {
        ALOGV("isOffloadSupported: has_video == true, returning false");
        return false;
    }

    //If duration is less than minimum value defined in property, return false
    if (property_get("audio.offload.min.duration.secs", propValue, NULL)) {
        if (offloadInfo.duration_us < (atoi(propValue) * 1000000 )) {
            ALOGV("Offload denied by duration < audio.offload.min.duration.secs(=%s)", propValue);
            return false;
        }
    } else if (offloadInfo.duration_us < OFFLOAD_DEFAULT_MIN_DURATION_SECS * 1000000) {
        ALOGV("Offload denied by duration < default min(=%u)", OFFLOAD_DEFAULT_MIN_DURATION_SECS);
        return false;
    }

    // Do not allow offloading if one non offloadable effect is enabled. This prevents from
    // creating an offloaded track and tearing it down immediately after start when audioflinger
    // detects there is an active non offloadable effect.
    // FIXME: We should check the audio session here but we do not have it in this context.
    // This may prevent offloading in rare situations where effects are left active by apps
    // in the background.
    if (mEffects.isNonOffloadableEffectEnabled()) {
        return false;
    }


#ifdef MTK_AUDIO
    // Do not allow offloading for MTK nonoffloadable effect
    String8 ret;
    ret = mpClientInterface->getParameters(AUDIO_IO_HANDLE_NONE,
                            String8("GetMusicPlusStatus"));
    if(ret == String8("GetMusicPlusStatus=1")) {
         return false;
    }
#endif

    // See if there is a profile to support this.
    // AUDIO_DEVICE_NONE
    sp<IOProfile> profile = getProfileForDirectOutput(AUDIO_DEVICE_NONE /*ignore device */,
                                            offloadInfo.sample_rate,
                                            offloadInfo.format,
                                            offloadInfo.channel_mask,
                                            AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD);
    ALOGV("isOffloadSupported() profile %sfound", profile != 0 ? "" : "NOT ");
    return (profile != 0);
}

status_t AudioPolicyManager::listAudioPorts(audio_port_role_t role,
                                            audio_port_type_t type,
                                            unsigned int *num_ports,
                                            struct audio_port *ports,
                                            unsigned int *generation)
{
    if (num_ports == NULL || (*num_ports != 0 && ports == NULL) ||
            generation == NULL) {
        return BAD_VALUE;
    }
#ifdef MTK_AUDIO
    MTK_ALOGV("listAudioPorts() role %d type %d num_ports %d ports %p", role, type, *num_ports, ports);
#else
    ALOGV("listAudioPorts() role %d type %d num_ports %d ports %p", role, type, *num_ports, ports);
#endif
    if (ports == NULL) {
        *num_ports = 0;
    }

    size_t portsWritten = 0;
    size_t portsMax = *num_ports;
    *num_ports = 0;
    if (type == AUDIO_PORT_TYPE_NONE || type == AUDIO_PORT_TYPE_DEVICE) {
        // do not report devices with type AUDIO_DEVICE_IN_STUB or AUDIO_DEVICE_OUT_STUB
        // as they are used by stub HALs by convention
        if (role == AUDIO_PORT_ROLE_SINK || role == AUDIO_PORT_ROLE_NONE) {
            for (size_t i = 0; i < mAvailableOutputDevices.size(); i++) {
                if (mAvailableOutputDevices[i]->type() == AUDIO_DEVICE_OUT_STUB) {
                    continue;
                }
                if (portsWritten < portsMax) {
                    mAvailableOutputDevices[i]->toAudioPort(&ports[portsWritten++]);
                }
                (*num_ports)++;
            }
        }
        if (role == AUDIO_PORT_ROLE_SOURCE || role == AUDIO_PORT_ROLE_NONE) {
            for (size_t i = 0; i < mAvailableInputDevices.size(); i++) {
                if (mAvailableInputDevices[i]->type() == AUDIO_DEVICE_IN_STUB) {
                    continue;
                }
                if (portsWritten < portsMax) {
                    mAvailableInputDevices[i]->toAudioPort(&ports[portsWritten++]);
                }
                (*num_ports)++;
            }
        }
    }
    if (type == AUDIO_PORT_TYPE_NONE || type == AUDIO_PORT_TYPE_MIX) {
        if (role == AUDIO_PORT_ROLE_SINK || role == AUDIO_PORT_ROLE_NONE) {
            for (size_t i = 0; i < mInputs.size() && portsWritten < portsMax; i++) {
                mInputs[i]->toAudioPort(&ports[portsWritten++]);
            }
            *num_ports += mInputs.size();
        }
        if (role == AUDIO_PORT_ROLE_SOURCE || role == AUDIO_PORT_ROLE_NONE) {
            size_t numOutputs = 0;
            for (size_t i = 0; i < mOutputs.size(); i++) {
                if (!mOutputs[i]->isDuplicated()) {
                    numOutputs++;
                    if (portsWritten < portsMax) {
                        mOutputs[i]->toAudioPort(&ports[portsWritten++]);
                    }
                }
            }
            *num_ports += numOutputs;
        }
    }
    *generation = curAudioPortGeneration();
#ifdef MTK_AUDIO
    MTK_ALOGV("listAudioPorts() got %zu ports needed %d", portsWritten, *num_ports);
#else
    ALOGV("listAudioPorts() got %zu ports needed %d", portsWritten, *num_ports);
#endif
    return NO_ERROR;
}

status_t AudioPolicyManager::getAudioPort(struct audio_port *port __unused)
{
    return NO_ERROR;
}

status_t AudioPolicyManager::createAudioPatch(const struct audio_patch *patch,
                                               audio_patch_handle_t *handle,
                                               uid_t uid)
{
    ALOGV("createAudioPatch()");

    if (handle == NULL || patch == NULL) {
        return BAD_VALUE;
    }
    ALOGV("createAudioPatch() num sources %d num sinks %d", patch->num_sources, patch->num_sinks);

    if (patch->num_sources == 0 || patch->num_sources > AUDIO_PATCH_PORTS_MAX ||
            patch->num_sinks == 0 || patch->num_sinks > AUDIO_PATCH_PORTS_MAX) {
        return BAD_VALUE;
    }
    // only one source per audio patch supported for now
    if (patch->num_sources > 1) {
        return INVALID_OPERATION;
    }

    if (patch->sources[0].role != AUDIO_PORT_ROLE_SOURCE) {
        return INVALID_OPERATION;
    }
    for (size_t i = 0; i < patch->num_sinks; i++) {
        if (patch->sinks[i].role != AUDIO_PORT_ROLE_SINK) {
            return INVALID_OPERATION;
        }
    }

    sp<AudioPatch> patchDesc;
    ssize_t index = mAudioPatches.indexOfKey(*handle);
#ifdef MTK_AUDIO    //  Used to first playback first sampling rate for FM
    uint32_t dSamplingRate = 44100;
    bool isUpdateSamplingRate = false;
#endif

    ALOGV("createAudioPatch source id %d role %d type %d", patch->sources[0].id,
                                                           patch->sources[0].role,
                                                           patch->sources[0].type);
#if LOG_NDEBUG == 0
    for (size_t i = 0; i < patch->num_sinks; i++) {
        ALOGV("createAudioPatch sink %zu: id %d role %d type %d", i, patch->sinks[i].id,
                                                             patch->sinks[i].role,
                                                             patch->sinks[i].type);
    }
#endif

    if (index >= 0) {
        patchDesc = mAudioPatches.valueAt(index);
        ALOGV("createAudioPatch() mUidCached %d patchDesc->mUid %d uid %d",
                                                                  mUidCached, patchDesc->mUid, uid);
        if (patchDesc->mUid != mUidCached && uid != patchDesc->mUid) {
            return INVALID_OPERATION;
        }
    } else {
        *handle = AUDIO_PATCH_HANDLE_NONE;
    }

    if (patch->sources[0].type == AUDIO_PORT_TYPE_MIX) {
        sp<SwAudioOutputDescriptor> outputDesc = mOutputs.getOutputFromId(patch->sources[0].id);
        if (outputDesc == NULL) {
            ALOGV("createAudioPatch() output not found for id %d", patch->sources[0].id);
            return BAD_VALUE;
        }
        ALOG_ASSERT(!outputDesc->isDuplicated(),"duplicated output %d in source in ports",
                                                outputDesc->mIoHandle);
        if (patchDesc != 0) {
            if (patchDesc->mPatch.sources[0].id != patch->sources[0].id) {
                ALOGV("createAudioPatch() source id differs for patch current id %d new id %d",
                                          patchDesc->mPatch.sources[0].id, patch->sources[0].id);
                return BAD_VALUE;
            }
        }
        DeviceVector devices;
        for (size_t i = 0; i < patch->num_sinks; i++) {
            // Only support mix to devices connection
            // TODO add support for mix to mix connection
            if (patch->sinks[i].type != AUDIO_PORT_TYPE_DEVICE) {
                ALOGV("createAudioPatch() source mix but sink is not a device");
                return INVALID_OPERATION;
            }
            sp<DeviceDescriptor> devDesc =
                    mAvailableOutputDevices.getDeviceFromId(patch->sinks[i].id);
            if (devDesc == 0) {
                ALOGV("createAudioPatch() out device not found for id %d", patch->sinks[i].id);
                return BAD_VALUE;
            }

            if (!outputDesc->mProfile->isCompatibleProfile(devDesc->type(),
                                                           devDesc->mAddress,
                                                           patch->sources[0].sample_rate,
                                                           NULL,  // updatedSamplingRate
                                                           patch->sources[0].format,
                                                           NULL,  // updatedFormat
                                                           patch->sources[0].channel_mask,
                                                           NULL,  // updatedChannelMask
                                                           AUDIO_OUTPUT_FLAG_NONE /*FIXME*/)) {
                ALOGV("createAudioPatch() profile not supported for device %08x",
                        devDesc->type());
                return INVALID_OPERATION;
            }
            devices.add(devDesc);
        }
        if (devices.size() == 0) {
            return INVALID_OPERATION;
        }

        // TODO: reconfigure output format and channels here
        ALOGV("createAudioPatch() setting device %08x on output %d",
              devices.types(), outputDesc->mIoHandle);
        setOutputDevice(outputDesc, devices.types(), true, 0, handle);
        index = mAudioPatches.indexOfKey(*handle);
        if (index >= 0) {
            if (patchDesc != 0 && patchDesc != mAudioPatches.valueAt(index)) {
                ALOGW("createAudioPatch() setOutputDevice() did not reuse the patch provided");
            }
            patchDesc = mAudioPatches.valueAt(index);
            patchDesc->mUid = uid;
            ALOGV("createAudioPatch() success");
        } else {
            ALOGW("createAudioPatch() setOutputDevice() failed to create a patch");
            return INVALID_OPERATION;
        }
    } else if (patch->sources[0].type == AUDIO_PORT_TYPE_DEVICE) {
        if (patch->sinks[0].type == AUDIO_PORT_TYPE_MIX) {
            // input device to input mix connection
            // only one sink supported when connecting an input device to a mix
            if (patch->num_sinks > 1) {
                return INVALID_OPERATION;
            }
            sp<AudioInputDescriptor> inputDesc = mInputs.getInputFromId(patch->sinks[0].id);
            if (inputDesc == NULL) {
                return BAD_VALUE;
            }
            if (patchDesc != 0) {
                if (patchDesc->mPatch.sinks[0].id != patch->sinks[0].id) {
                    return BAD_VALUE;
                }
            }
            sp<DeviceDescriptor> devDesc =
                    mAvailableInputDevices.getDeviceFromId(patch->sources[0].id);
            if (devDesc == 0) {
                return BAD_VALUE;
            }

            if (!inputDesc->mProfile->isCompatibleProfile(devDesc->type(),
                                                          devDesc->mAddress,
                                                          patch->sinks[0].sample_rate,
                                                          NULL, /*updatedSampleRate*/
                                                          patch->sinks[0].format,
                                                          NULL, /*updatedFormat*/
                                                          patch->sinks[0].channel_mask,
                                                          NULL, /*updatedChannelMask*/
                                                          // FIXME for the parameter type,
                                                          // and the NONE
                                                          (audio_output_flags_t)
                                                            AUDIO_INPUT_FLAG_NONE)) {
                return INVALID_OPERATION;
            }
            // TODO: reconfigure output format and channels here
            ALOGV("createAudioPatch() setting device %08x on output %d",
                                                  devDesc->type(), inputDesc->mIoHandle);
            setInputDevice(inputDesc->mIoHandle, devDesc->type(), true, handle);
            index = mAudioPatches.indexOfKey(*handle);
            if (index >= 0) {
                if (patchDesc != 0 && patchDesc != mAudioPatches.valueAt(index)) {
                    ALOGW("createAudioPatch() setInputDevice() did not reuse the patch provided");
                }
                patchDesc = mAudioPatches.valueAt(index);
                patchDesc->mUid = uid;
                ALOGV("createAudioPatch() success");
            } else {
                ALOGW("createAudioPatch() setInputDevice() failed to create a patch");
                return INVALID_OPERATION;
            }
        } else if (patch->sinks[0].type == AUDIO_PORT_TYPE_DEVICE) {
            // device to device connection
            if (patchDesc != 0) {
                if (patchDesc->mPatch.sources[0].id != patch->sources[0].id) {
                    return BAD_VALUE;
                }
            }
            sp<DeviceDescriptor> srcDeviceDesc =
                    mAvailableInputDevices.getDeviceFromId(patch->sources[0].id);
            if (srcDeviceDesc == 0) {
                return BAD_VALUE;
            }

            //update source and sink with our own data as the data passed in the patch may
            // be incomplete.
            struct audio_patch newPatch = *patch;
            srcDeviceDesc->toAudioPortConfig(&newPatch.sources[0], &patch->sources[0]);

            for (size_t i = 0; i < patch->num_sinks; i++) {
                if (patch->sinks[i].type != AUDIO_PORT_TYPE_DEVICE) {
                    ALOGV("createAudioPatch() source device but one sink is not a device");
                    return INVALID_OPERATION;
                }

                sp<DeviceDescriptor> sinkDeviceDesc =
                        mAvailableOutputDevices.getDeviceFromId(patch->sinks[i].id);
                if (sinkDeviceDesc == 0) {
                    return BAD_VALUE;
                }
                sinkDeviceDesc->toAudioPortConfig(&newPatch.sinks[i], &patch->sinks[i]);
#ifdef MTK_AUDIO    //  Used to first playback first sampling rate for FM
                if (mPrimaryOutput != 0 && isUpdateSamplingRate == false) {
                    MTK_ALOGV("support dev %x  sink[%zu] dev %x",mPrimaryOutput->supportedDevices(),i,sinkDeviceDesc->type());
                    if (mPrimaryOutput->supportedDevices() & sinkDeviceDesc->type()) {
                        int dTempSamplingRate;
                        dTempSamplingRate = (int) patch->sinks[i].sample_rate;
                        SampleRateRequestFocus(mPrimaryOutput->mIoHandle, AUDIO_STREAM_MUSIC, &dTempSamplingRate);
                        newPatch.sinks[i].sample_rate = dSamplingRate = (uint32_t) dTempSamplingRate;
                        //dSamplingRate = newPatch.sinks[i].sample_rate = getSuitSampleRateForAudioPatch(patch->sinks[i].sample_rate);
                        isUpdateSamplingRate = true;
                    }
                }
#endif

                // create a software bridge in PatchPanel if:
                // - source and sink devices are on differnt HW modules OR
                // - audio HAL version is < 3.0
                if ((srcDeviceDesc->getModuleHandle() != sinkDeviceDesc->getModuleHandle()) ||
                        (srcDeviceDesc->mModule->getHalVersion() < AUDIO_DEVICE_API_VERSION_3_0)) {
                    // support only one sink device for now to simplify output selection logic
                    if (patch->num_sinks > 1) {
                        return INVALID_OPERATION;
                    }
                    SortedVector<audio_io_handle_t> outputs =
                                            getOutputsForDevice(sinkDeviceDesc->type(), mOutputs);
                    // if the sink device is reachable via an opened output stream, request to go via
                    // this output stream by adding a second source to the patch description
                    audio_io_handle_t output = selectOutput(outputs,
                                                            AUDIO_OUTPUT_FLAG_NONE,
                                                            AUDIO_FORMAT_INVALID);
                    if (output != AUDIO_IO_HANDLE_NONE) {
                        sp<AudioOutputDescriptor> outputDesc = mOutputs.valueFor(output);
                        if (outputDesc->isDuplicated()) {
                            return INVALID_OPERATION;
                        }
                        outputDesc->toAudioPortConfig(&newPatch.sources[1], &patch->sources[0]);
                        newPatch.sources[1].ext.mix.usecase.stream = AUDIO_STREAM_PATCH;
                        newPatch.num_sources = 2;
                    }
                }
            }
            // TODO: check from routing capabilities in config file and other conflicting patches
#ifdef MTK_AUDIO    //  Used to first playback first sampling rate for FM
            if (isUpdateSamplingRate) {
                AddSampleRateArray(AUDIO_STREAM_MUSIC, dSamplingRate);
                SampleRatePolicy(AUDIO_STREAM_MUSIC, dSamplingRate);
            }
#endif

            audio_patch_handle_t afPatchHandle = AUDIO_PATCH_HANDLE_NONE;
            if (index >= 0) {
                afPatchHandle = patchDesc->mAfPatchHandle;
            }

            status_t status = mpClientInterface->createAudioPatch(&newPatch,
                                                                  &afPatchHandle,
                                                                  0);
            ALOGV("createAudioPatch() patch panel returned %d patchHandle %d",
                                                                  status, afPatchHandle);
            if (status == NO_ERROR) {
                if (index < 0) {
                    patchDesc = new AudioPatch(&newPatch, uid);
                    addAudioPatch(patchDesc->mHandle, patchDesc);
                } else {
                    patchDesc->mPatch = newPatch;
                }
                patchDesc->mAfPatchHandle = afPatchHandle;
                *handle = patchDesc->mHandle;
                nextAudioPortGeneration();
                mpClientInterface->onAudioPatchListUpdate();
            } else {
                ALOGW("createAudioPatch() patch panel could not connect device patch, error %d",
                status);
#ifdef MTK_AUDIO    //  Used to first playback first sampling rate for FM
                if (isUpdateSamplingRate) {
                    RemoveSampleRateArray(AUDIO_STREAM_MUSIC, dSamplingRate);
                    if (CheckStreamActive() == false) {
                        ReleaseFMIndirectMode(dSamplingRate);
                        PolicyRestore();
                    }
                    SampleRateUnrequestFocus(mPrimaryOutput->mIoHandle, AUDIO_STREAM_MUSIC, dSamplingRate);
                }
#endif
                return INVALID_OPERATION;
            }
        } else {
            return BAD_VALUE;
        }
    } else {
        return BAD_VALUE;
    }
    return NO_ERROR;
}

status_t AudioPolicyManager::releaseAudioPatch(audio_patch_handle_t handle,
                                                  uid_t uid)
{
    ALOGV("releaseAudioPatch() patch %d", handle);

    ssize_t index = mAudioPatches.indexOfKey(handle);

    if (index < 0) {
        return BAD_VALUE;
    }
    sp<AudioPatch> patchDesc = mAudioPatches.valueAt(index);
    ALOGV("releaseAudioPatch() mUidCached %d patchDesc->mUid %d uid %d",
          mUidCached, patchDesc->mUid, uid);
    if (patchDesc->mUid != mUidCached && uid != patchDesc->mUid) {
        return INVALID_OPERATION;
    }

    struct audio_patch *patch = &patchDesc->mPatch;
    patchDesc->mUid = mUidCached;
    if (patch->sources[0].type == AUDIO_PORT_TYPE_MIX) {
        sp<SwAudioOutputDescriptor> outputDesc = mOutputs.getOutputFromId(patch->sources[0].id);
        if (outputDesc == NULL) {
            ALOGV("releaseAudioPatch() output not found for id %d", patch->sources[0].id);
            return BAD_VALUE;
        }

        setOutputDevice(outputDesc,
                        getNewOutputDevice(outputDesc, true /*fromCache*/),
                       true,
                       0,
                       NULL);
    } else if (patch->sources[0].type == AUDIO_PORT_TYPE_DEVICE) {
        if (patch->sinks[0].type == AUDIO_PORT_TYPE_MIX) {
            sp<AudioInputDescriptor> inputDesc = mInputs.getInputFromId(patch->sinks[0].id);
            if (inputDesc == NULL) {
                ALOGV("releaseAudioPatch() input not found for id %d", patch->sinks[0].id);
                return BAD_VALUE;
            }
            setInputDevice(inputDesc->mIoHandle,
                           getNewInputDevice(inputDesc->mIoHandle),
                           true,
                           NULL);
        } else if (patch->sinks[0].type == AUDIO_PORT_TYPE_DEVICE) {
            audio_patch_handle_t afPatchHandle = patchDesc->mAfPatchHandle;
            status_t status = mpClientInterface->releaseAudioPatch(patchDesc->mAfPatchHandle, 0);
            ALOGV("releaseAudioPatch() patch panel returned %d patchHandle %d",
                                                              status, patchDesc->mAfPatchHandle);
            removeAudioPatch(patchDesc->mHandle);
            nextAudioPortGeneration();
            mpClientInterface->onAudioPatchListUpdate();
        } else {
            return BAD_VALUE;
        }
    } else {
        return BAD_VALUE;
    }
#ifdef MTK_AUDIO
    uint32_t dSamplingRate;
    if (patch->sources[0].type == AUDIO_PORT_TYPE_MIX) {
        /* doesn't support now
        dSamplingRate = patch->sinks[0].sample_rate;
        RemoveSampleRateArray(AUDIO_STREAM_MUSIC,dSamplingRate);
        if (CheckStreamActive() == false) {
            ReleaseFMIndirectMode(dSamplingRate);
            PolicyRestore();
        }
        */
    } else if (patch->sources[0].type == AUDIO_PORT_TYPE_DEVICE) {
        if (patch->sinks[0].type == AUDIO_PORT_TYPE_MIX) {
        // do nothing
        } else if (patch->sinks[0].type == AUDIO_PORT_TYPE_DEVICE) {
            if (mPrimaryOutput != 0) {
                    sp<DeviceDescriptor> sinkDeviceDesc =
                    mAvailableOutputDevices.getDeviceFromId(patch->sinks[0].id);
                    if (sinkDeviceDesc == 0) {
                        ALOGE("DeviceErr, but we recovery it");
                    } else {
                        if (mPrimaryOutput->supportedDevices() & sinkDeviceDesc->type()) {
                            ALOGV("del dev 0x%x", sinkDeviceDesc->type());
                        }
                    }
                    int dTempSamplingRate;
                    dSamplingRate = patch->sinks[0].sample_rate;
                    RemoveSampleRateArray(AUDIO_STREAM_MUSIC, dSamplingRate);
                    if (CheckStreamActive() == false) {
                        ReleaseFMIndirectMode(dSamplingRate);
                        PolicyRestore();
                    }
                    dTempSamplingRate = (int) dSamplingRate;
                    SampleRateUnrequestFocus(mPrimaryOutput->mIoHandle, AUDIO_STREAM_MUSIC, dTempSamplingRate);
            }
        }
    }
#endif
    return NO_ERROR;
}

status_t AudioPolicyManager::listAudioPatches(unsigned int *num_patches,
                                              struct audio_patch *patches,
                                              unsigned int *generation)
{
    if (generation == NULL) {
        return BAD_VALUE;
    }
    *generation = curAudioPortGeneration();
    return mAudioPatches.listAudioPatches(num_patches, patches);
}

status_t AudioPolicyManager::setAudioPortConfig(const struct audio_port_config *config)
{
    ALOGV("setAudioPortConfig()");

    if (config == NULL) {
        return BAD_VALUE;
    }
    ALOGV("setAudioPortConfig() on port handle %d", config->id);
    // Only support gain configuration for now
    if (config->config_mask != AUDIO_PORT_CONFIG_GAIN) {
        return INVALID_OPERATION;
    }

    sp<AudioPortConfig> audioPortConfig;
    if (config->type == AUDIO_PORT_TYPE_MIX) {
        if (config->role == AUDIO_PORT_ROLE_SOURCE) {
            sp<SwAudioOutputDescriptor> outputDesc = mOutputs.getOutputFromId(config->id);
            if (outputDesc == NULL) {
                return BAD_VALUE;
            }
            ALOG_ASSERT(!outputDesc->isDuplicated(),
                        "setAudioPortConfig() called on duplicated output %d",
                        outputDesc->mIoHandle);
            audioPortConfig = outputDesc;
        } else if (config->role == AUDIO_PORT_ROLE_SINK) {
            sp<AudioInputDescriptor> inputDesc = mInputs.getInputFromId(config->id);
            if (inputDesc == NULL) {
                return BAD_VALUE;
            }
            audioPortConfig = inputDesc;
        } else {
            return BAD_VALUE;
        }
    } else if (config->type == AUDIO_PORT_TYPE_DEVICE) {
        sp<DeviceDescriptor> deviceDesc;
        if (config->role == AUDIO_PORT_ROLE_SOURCE) {
            deviceDesc = mAvailableInputDevices.getDeviceFromId(config->id);
        } else if (config->role == AUDIO_PORT_ROLE_SINK) {
            deviceDesc = mAvailableOutputDevices.getDeviceFromId(config->id);
        } else {
            return BAD_VALUE;
        }
        if (deviceDesc == NULL) {
            return BAD_VALUE;
        }
        audioPortConfig = deviceDesc;
    } else {
        return BAD_VALUE;
    }

    struct audio_port_config backupConfig;
    status_t status = audioPortConfig->applyAudioPortConfig(config, &backupConfig);
    if (status == NO_ERROR) {
        struct audio_port_config newConfig;
        audioPortConfig->toAudioPortConfig(&newConfig, config);
        status = mpClientInterface->setAudioPortConfig(&newConfig, 0);
    }
    if (status != NO_ERROR) {
        audioPortConfig->applyAudioPortConfig(&backupConfig);
    }

    return status;
}

void AudioPolicyManager::releaseResourcesForUid(uid_t uid)
{
    clearAudioSources(uid);
    clearAudioPatches(uid);
    clearSessionRoutes(uid);
}

void AudioPolicyManager::clearAudioPatches(uid_t uid)
{
    for (ssize_t i = (ssize_t)mAudioPatches.size() - 1; i >= 0; i--)  {
        sp<AudioPatch> patchDesc = mAudioPatches.valueAt(i);
        if (patchDesc->mUid == uid) {
            releaseAudioPatch(mAudioPatches.keyAt(i), uid);
        }
    }
}

void AudioPolicyManager::checkStrategyRoute(routing_strategy strategy,
                                            audio_io_handle_t ouptutToSkip)
{
    audio_devices_t device = getDeviceForStrategy(strategy, false /*fromCache*/);
    SortedVector<audio_io_handle_t> outputs = getOutputsForDevice(device, mOutputs);
    for (size_t j = 0; j < mOutputs.size(); j++) {
        if (mOutputs.keyAt(j) == ouptutToSkip) {
            continue;
        }
        sp<SwAudioOutputDescriptor> outputDesc = mOutputs.valueAt(j);
        if (!isStrategyActive(outputDesc, (routing_strategy)strategy)) {
            continue;
        }
        // If the default device for this strategy is on another output mix,
        // invalidate all tracks in this strategy to force re connection.
        // Otherwise select new device on the output mix.
        if (outputs.indexOf(mOutputs.keyAt(j)) < 0) {
            for (int stream = 0; stream < AUDIO_STREAM_FOR_POLICY_CNT; stream++) {
                if (getStrategy((audio_stream_type_t)stream) == strategy) {
                    mpClientInterface->invalidateStream((audio_stream_type_t)stream);
                }
            }
        } else {
            audio_devices_t newDevice = getNewOutputDevice(outputDesc, false /*fromCache*/);
            setOutputDevice(outputDesc, newDevice, false);
        }
    }
}

void AudioPolicyManager::clearSessionRoutes(uid_t uid)
{
    // remove output routes associated with this uid
    SortedVector<routing_strategy> affectedStrategies;
    for (ssize_t i = (ssize_t)mOutputRoutes.size() - 1; i >= 0; i--)  {
        sp<SessionRoute> route = mOutputRoutes.valueAt(i);
        if (route->mUid == uid) {
            mOutputRoutes.removeItemsAt(i);
            if (route->mDeviceDescriptor != 0) {
                affectedStrategies.add(getStrategy(route->mStreamType));
            }
        }
    }
    // reroute outputs if necessary
    for (size_t i = 0; i < affectedStrategies.size(); i++) {
        checkStrategyRoute(affectedStrategies[i], AUDIO_IO_HANDLE_NONE);
    }

    // remove input routes associated with this uid
    SortedVector<audio_source_t> affectedSources;
    for (ssize_t i = (ssize_t)mInputRoutes.size() - 1; i >= 0; i--)  {
        sp<SessionRoute> route = mInputRoutes.valueAt(i);
        if (route->mUid == uid) {
            mInputRoutes.removeItemsAt(i);
            if (route->mDeviceDescriptor != 0) {
                affectedSources.add(route->mSource);
            }
        }
    }
    // reroute inputs if necessary
    SortedVector<audio_io_handle_t> inputsToClose;
    for (size_t i = 0; i < mInputs.size(); i++) {
        sp<AudioInputDescriptor> inputDesc = mInputs.valueAt(i);
        if (affectedSources.indexOf(inputDesc->inputSource()) >= 0) {
            inputsToClose.add(inputDesc->mIoHandle);
        }
    }
    for (size_t i = 0; i < inputsToClose.size(); i++) {
        closeInput(inputsToClose[i]);
    }
}

void AudioPolicyManager::clearAudioSources(uid_t uid)
{
    for (ssize_t i = (ssize_t)mAudioSources.size() - 1; i >= 0; i--)  {
        sp<AudioSourceDescriptor> sourceDesc = mAudioSources.valueAt(i);
        if (sourceDesc->mUid == uid) {
            stopAudioSource(mAudioSources.keyAt(i));
        }
    }
}

status_t AudioPolicyManager::acquireSoundTriggerSession(audio_session_t *session,
                                       audio_io_handle_t *ioHandle,
                                       audio_devices_t *device)
{
    *session = (audio_session_t)mpClientInterface->newAudioUniqueId(AUDIO_UNIQUE_ID_USE_SESSION);
    *ioHandle = (audio_io_handle_t)mpClientInterface->newAudioUniqueId(AUDIO_UNIQUE_ID_USE_INPUT);
    *device = getDeviceAndMixForInputSource(AUDIO_SOURCE_HOTWORD);

    return mSoundTriggerSessions.acquireSession(*session, *ioHandle);
}

status_t AudioPolicyManager::startAudioSource(const struct audio_port_config *source,
                                  const audio_attributes_t *attributes,
                                  audio_io_handle_t *handle,
                                  uid_t uid)
{
    ALOGV("%s source %p attributes %p handle %p", __FUNCTION__, source, attributes, handle);
    if (source == NULL || attributes == NULL || handle == NULL) {
        return BAD_VALUE;
    }

    *handle = AUDIO_IO_HANDLE_NONE;

    if (source->role != AUDIO_PORT_ROLE_SOURCE ||
            source->type != AUDIO_PORT_TYPE_DEVICE) {
        ALOGV("%s INVALID_OPERATION source->role %d source->type %d", __FUNCTION__, source->role, source->type);
        return INVALID_OPERATION;
    }

    sp<DeviceDescriptor> srcDeviceDesc =
            mAvailableInputDevices.getDevice(source->ext.device.type,
                                              String8(source->ext.device.address));
    if (srcDeviceDesc == 0) {
        ALOGV("%s source->ext.device.type %08x not found", __FUNCTION__, source->ext.device.type);
        return BAD_VALUE;
    }
    sp<AudioSourceDescriptor> sourceDesc =
            new AudioSourceDescriptor(srcDeviceDesc, attributes, uid);

    struct audio_patch dummyPatch;
    sp<AudioPatch> patchDesc = new AudioPatch(&dummyPatch, uid);
    sourceDesc->mPatchDesc = patchDesc;

    status_t status = connectAudioSource(sourceDesc);
    if (status == NO_ERROR) {
        mAudioSources.add(sourceDesc->getHandle(), sourceDesc);
        *handle = sourceDesc->getHandle();
    }
    return status;
}

status_t AudioPolicyManager::connectAudioSource(const sp<AudioSourceDescriptor>& sourceDesc)
{
    ALOGV("%s handle %d", __FUNCTION__, sourceDesc->getHandle());

    // make sure we only have one patch per source.
    disconnectAudioSource(sourceDesc);

    routing_strategy strategy = (routing_strategy) getStrategyForAttr(&sourceDesc->mAttributes);
    audio_stream_type_t stream = streamTypefromAttributesInt(&sourceDesc->mAttributes);
    sp<DeviceDescriptor> srcDeviceDesc = sourceDesc->mDevice;

    audio_devices_t sinkDevice = getDeviceForStrategy(strategy, true);
    sp<DeviceDescriptor> sinkDeviceDesc =
            mAvailableOutputDevices.getDevice(sinkDevice, String8(""));

    audio_patch_handle_t afPatchHandle = AUDIO_PATCH_HANDLE_NONE;
    struct audio_patch *patch = &sourceDesc->mPatchDesc->mPatch;

    if (srcDeviceDesc->getAudioPort()->mModule->getHandle() ==
            sinkDeviceDesc->getAudioPort()->mModule->getHandle() &&
            srcDeviceDesc->getAudioPort()->mModule->getHalVersion() >= AUDIO_DEVICE_API_VERSION_3_0 &&
            srcDeviceDesc->getAudioPort()->mGains.size() > 0) {
        ALOGV("%s AUDIO_DEVICE_API_VERSION_3_0", __FUNCTION__);
        //   create patch between src device and output device
        //   create Hwoutput and add to mHwOutputs
    } else {
        SortedVector<audio_io_handle_t> outputs = getOutputsForDevice(sinkDevice, mOutputs);
        audio_io_handle_t output =
                selectOutput(outputs, AUDIO_OUTPUT_FLAG_NONE, AUDIO_FORMAT_INVALID);
        if (output == AUDIO_IO_HANDLE_NONE) {
            ALOGV("%s no output for device %08x", __FUNCTION__, sinkDevice);
            return INVALID_OPERATION;
        }
        sp<SwAudioOutputDescriptor> outputDesc = mOutputs.valueFor(output);
        if (outputDesc->isDuplicated()) {
            ALOGV("%s output for device %08x is duplicated", __FUNCTION__, sinkDevice);
            return INVALID_OPERATION;
        }
        // create a special patch with no sink and two sources:
        // - the second source indicates to PatchPanel through which output mix this patch should
        // be connected as well as the stream type for volume control
        // - the sink is defined by whatever output device is currently selected for the output
        // though which this patch is routed.
        patch->num_sinks = 0;
        patch->num_sources = 2;
        srcDeviceDesc->toAudioPortConfig(&patch->sources[0], NULL);
        outputDesc->toAudioPortConfig(&patch->sources[1], NULL);
        patch->sources[1].ext.mix.usecase.stream = stream;
        status_t status = mpClientInterface->createAudioPatch(patch,
                                                              &afPatchHandle,
                                                              0);
        ALOGV("%s patch panel returned %d patchHandle %d", __FUNCTION__,
                                                              status, afPatchHandle);
        if (status != NO_ERROR) {
            ALOGW("%s patch panel could not connect device patch, error %d",
                  __FUNCTION__, status);
            return INVALID_OPERATION;
        }
        uint32_t delayMs = 0;
        status = startSource(outputDesc, stream, sinkDevice, NULL, &delayMs);

        if (status != NO_ERROR) {
            mpClientInterface->releaseAudioPatch(sourceDesc->mPatchDesc->mAfPatchHandle, 0);
            return status;
        }
        sourceDesc->mSwOutput = outputDesc;
        if (delayMs != 0) {
            usleep(delayMs * 1000);
        }
    }

    sourceDesc->mPatchDesc->mAfPatchHandle = afPatchHandle;
    addAudioPatch(sourceDesc->mPatchDesc->mHandle, sourceDesc->mPatchDesc);

    return NO_ERROR;
}

status_t AudioPolicyManager::stopAudioSource(audio_io_handle_t handle __unused)
{
    sp<AudioSourceDescriptor> sourceDesc = mAudioSources.valueFor(handle);
    ALOGV("%s handle %d", __FUNCTION__, handle);
    if (sourceDesc == 0) {
        ALOGW("%s unknown source for handle %d", __FUNCTION__, handle);
        return BAD_VALUE;
    }
    status_t status = disconnectAudioSource(sourceDesc);

    mAudioSources.removeItem(handle);
    return status;
}

status_t AudioPolicyManager::setMasterMono(bool mono)
{
    if (mMasterMono == mono) {
        return NO_ERROR;
    }
    mMasterMono = mono;
    // if enabling mono we close all offloaded devices, which will invalidate the
    // corresponding AudioTrack. The AudioTrack client/MediaPlayer is responsible
    // for recreating the new AudioTrack as non-offloaded PCM.
    //
    // If disabling mono, we leave all tracks as is: we don't know which clients
    // and tracks are able to be recreated as offloaded. The next "song" should
    // play back offloaded.
    if (mMasterMono) {
        Vector<audio_io_handle_t> offloaded;
        for (size_t i = 0; i < mOutputs.size(); ++i) {
            sp<SwAudioOutputDescriptor> desc = mOutputs.valueAt(i);
            if (desc->mFlags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) {
                offloaded.push(desc->mIoHandle);
            }
        }
        for (size_t i = 0; i < offloaded.size(); ++i) {
            closeOutput(offloaded[i]);
        }
    }
    // update master mono for all remaining outputs
    for (size_t i = 0; i < mOutputs.size(); ++i) {
        updateMono(mOutputs.keyAt(i));
    }
    return NO_ERROR;
}

status_t AudioPolicyManager::getMasterMono(bool *mono)
{
    *mono = mMasterMono;
    return NO_ERROR;
}

status_t AudioPolicyManager::disconnectAudioSource(const sp<AudioSourceDescriptor>& sourceDesc)
{
    ALOGV("%s handle %d", __FUNCTION__, sourceDesc->getHandle());

    sp<AudioPatch> patchDesc = mAudioPatches.valueFor(sourceDesc->mPatchDesc->mHandle);
    if (patchDesc == 0) {
        ALOGW("%s source has no patch with handle %d", __FUNCTION__,
              sourceDesc->mPatchDesc->mHandle);
        return BAD_VALUE;
    }
    removeAudioPatch(sourceDesc->mPatchDesc->mHandle);

    audio_stream_type_t stream = streamTypefromAttributesInt(&sourceDesc->mAttributes);
    sp<SwAudioOutputDescriptor> swOutputDesc = sourceDesc->mSwOutput.promote();
    if (swOutputDesc != 0) {
        stopSource(swOutputDesc, stream, false);
        mpClientInterface->releaseAudioPatch(patchDesc->mAfPatchHandle, 0);
    } else {
        sp<HwAudioOutputDescriptor> hwOutputDesc = sourceDesc->mHwOutput.promote();
        if (hwOutputDesc != 0) {
          //   release patch between src device and output device
          //   close Hwoutput and remove from mHwOutputs
        } else {
            ALOGW("%s source has neither SW nor HW output", __FUNCTION__);
        }
    }
    return NO_ERROR;
}

sp<AudioSourceDescriptor> AudioPolicyManager::getSourceForStrategyOnOutput(
        audio_io_handle_t output, routing_strategy strategy)
{
    sp<AudioSourceDescriptor> source;
    for (size_t i = 0; i < mAudioSources.size(); i++)  {
        sp<AudioSourceDescriptor> sourceDesc = mAudioSources.valueAt(i);
        routing_strategy sourceStrategy =
                (routing_strategy) getStrategyForAttr(&sourceDesc->mAttributes);
        sp<SwAudioOutputDescriptor> outputDesc = sourceDesc->mSwOutput.promote();
        if (sourceStrategy == strategy && outputDesc != 0 && outputDesc->mIoHandle == output) {
            source = sourceDesc;
            break;
        }
    }
    return source;
}

// ----------------------------------------------------------------------------
// AudioPolicyManager
// ----------------------------------------------------------------------------
uint32_t AudioPolicyManager::nextAudioPortGeneration()
{
    return android_atomic_inc(&mAudioPortGeneration);
}

AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
    :
#ifdef AUDIO_POLICY_TEST
    Thread(false),
#endif //AUDIO_POLICY_TEST
    mLimitRingtoneVolume(false), mLastVoiceVolume(-1.0f),
    mA2dpSuspended(false),
    mAudioPortGeneration(1),
    mBeaconMuteRefCount(0),
    mBeaconPlayingRefCount(0),
    mBeaconMuted(false),
    mTtsOutputAvailable(false),
    mMasterMono(false)
{
#ifdef MTK_AUDIO
    char value[PROPERTY_VALUE_MAX];
    property_get(PROPERTY_KEY_POLICY_DEBUG, value, "0");
    if (atoi(value)) {
        gMTKPolicyLoglevel = atoi(value);
    } else {
        gMTKPolicyLoglevel = 0;
    }
    ALOGD("+%s %d debug [%d]",__FUNCTION__,__LINE__,gMTKPolicyLoglevel);
    mNativeAutoDetectHeadSetFlag = false;
    mFMDirectAudioPatchEnable = false;
    mNeedRemapVoiceVolumeIndex = false;
#ifdef MTK_CROSSMOUNT_SUPPORT
    mAudioPolicyVendorControl.setCrossMountLocalPlayback(false);
    mAudioPolicyVendorControl.setCrossMountMicLocalPlayback(false);
    mAudioPolicyVendorControl.setCrossMountMicAudioMixerEnable(false);
#endif
#ifdef MTK_AUDIO_GAIN_TABLE
    LoadCustomVolume();
    mVolumeStream = -1;
    mVolumeIndex  = -1 ;
    mVolumeDevice = AUDIO_DEVICE_NONE;
#endif
    mTty_Ctm = AUD_TTY_OFF;
#endif
    mUidCached = getuid();
    mpClientInterface = clientInterface;

    // TODO: remove when legacy conf file is removed. true on devices that use DRC on the
    // DEVICE_CATEGORY_SPEAKER path to boost soft sounds, used to adjust volume curves accordingly.
    // Note: remove also speaker_drc_enabled from global configuration of XML config file.
    bool speakerDrcEnabled = false;

#ifdef USE_XML_AUDIO_POLICY_CONF
    mVolumeCurves = new VolumeCurvesCollection();
    AudioPolicyConfig config(mHwModules, mAvailableOutputDevices, mAvailableInputDevices,
                             mDefaultOutputDevice, speakerDrcEnabled,
                             static_cast<VolumeCurvesCollection *>(mVolumeCurves));
    PolicySerializer serializer;
    if (serializer.deserialize(AUDIO_POLICY_XML_CONFIG_FILE, config) != NO_ERROR)
#else
    mVolumeCurves = new StreamDescriptorCollection();
    AudioPolicyConfig config(mHwModules, mAvailableOutputDevices, mAvailableInputDevices,
                             mDefaultOutputDevice, speakerDrcEnabled);
    if ((ConfigParsingUtils::loadConfig(AUDIO_POLICY_VENDOR_CONFIG_FILE, config) != NO_ERROR) &&
            (ConfigParsingUtils::loadConfig(AUDIO_POLICY_CONFIG_FILE, config) != NO_ERROR))
#endif
    {
        ALOGE("could not load audio policy configuration file, setting defaults");
        config.setDefault();
    }
    // must be done after reading the policy (since conditionned by Speaker Drc Enabling)
    mVolumeCurves->initializeVolumeCurves(speakerDrcEnabled);

    // Once policy config has been parsed, retrieve an instance of the engine and initialize it.
    audio_policy::EngineInstance *engineInstance = audio_policy::EngineInstance::getInstance();
    if (!engineInstance) {
        ALOGE("%s:  Could not get an instance of policy engine", __FUNCTION__);
        return;
    }
    // Retrieve the Policy Manager Interface
    mEngine = engineInstance->queryInterface<AudioPolicyManagerInterface>();
    if (mEngine == NULL) {
        ALOGE("%s: Failed to get Policy Engine Interface", __FUNCTION__);
        return;
    }
    mEngine->setObserver(this);
    status_t status = mEngine->initCheck();
    ALOG_ASSERT(status == NO_ERROR, "Policy engine not initialized(err=%d)", status);

    // mAvailableOutputDevices and mAvailableInputDevices now contain all attached devices
    // open all output streams needed to access attached devices
    audio_devices_t outputDeviceTypes = mAvailableOutputDevices.types();
    audio_devices_t inputDeviceTypes = mAvailableInputDevices.types() & ~AUDIO_DEVICE_BIT_IN;
    for (size_t i = 0; i < mHwModules.size(); i++) {
        mHwModules[i]->mHandle = mpClientInterface->loadHwModule(mHwModules[i]->getName());
        if (mHwModules[i]->mHandle == 0) {
            ALOGW("could not open HW module %s", mHwModules[i]->getName());
            continue;
        }
        // open all output streams needed to access attached devices
        // except for direct output streams that are only opened when they are actually
        // required by an app.
        // This also validates mAvailableOutputDevices list
        for (size_t j = 0; j < mHwModules[i]->mOutputProfiles.size(); j++)
        {
            const sp<IOProfile> outProfile = mHwModules[i]->mOutputProfiles[j];

            if (!outProfile->hasSupportedDevices()) {
                ALOGW("Output profile contains no device on module %s", mHwModules[i]->getName());
                continue;
            }
            if ((outProfile->getFlags() & AUDIO_OUTPUT_FLAG_TTS) != 0) {
                mTtsOutputAvailable = true;
            }

            if ((outProfile->getFlags() & AUDIO_OUTPUT_FLAG_DIRECT) != 0) {
                continue;
            }
#ifdef MTK_AUDIO    //Skip SPK when speaker enable for Fast output
            if (AUDIO_OUTPUT_FLAG_FAST & outProfile->getFlags()) {
                String8 command = mpClientInterface->getParameters(0, String8("GetSpeakerProtection"));
                AudioParameter param = AudioParameter(command);
                int valueInt;
                if (param.getInt(String8("GetSpeakerProtection"), valueInt) == NO_ERROR &&
                    valueInt == 1) {
                    sp<DeviceDescriptor> devDesc = outProfile->getSupportedDeviceByAddress(AUDIO_DEVICE_OUT_SPEAKER, String8(""));
                    if (devDesc != NULL) {
                        if (outProfile->removeSupportedDevice(devDesc) >= 0) {
                            ALOGD("Remove SPK in Fast");
                        } else {
                            ALOGW("Remove SPK Fail in Fast");
                        }
                    } else {
                        ALOGW("Not support SPK in Fast");
                    }
                } else {
                    ALOGV("Don't need rm SPK in Fast");
                }
            }
#endif
            audio_devices_t profileType = outProfile->getSupportedDevicesType();
            if ((profileType & mDefaultOutputDevice->type()) != AUDIO_DEVICE_NONE) {
                profileType = mDefaultOutputDevice->type();
            } else {
                // chose first device present in profile's SupportedDevices also part of
                // outputDeviceTypes
                profileType = outProfile->getSupportedDeviceForType(outputDeviceTypes);
            }
            if ((profileType & outputDeviceTypes) == 0) {
                continue;
            }
            sp<SwAudioOutputDescriptor> outputDesc = new SwAudioOutputDescriptor(outProfile,
                                                                                 mpClientInterface);
            const DeviceVector &supportedDevices = outProfile->getSupportedDevices();
            const DeviceVector &devicesForType = supportedDevices.getDevicesFromType(profileType);
            String8 address = devicesForType.size() > 0 ? devicesForType.itemAt(0)->mAddress
                    : String8("");

            outputDesc->mDevice = profileType;
            audio_config_t config = AUDIO_CONFIG_INITIALIZER;
            config.sample_rate = outputDesc->mSamplingRate;
            config.channel_mask = outputDesc->mChannelMask;
            config.format = outputDesc->mFormat;
            audio_io_handle_t output = AUDIO_IO_HANDLE_NONE;
            status_t status = mpClientInterface->openOutput(outProfile->getModuleHandle(),
                                                            &output,
                                                            &config,
                                                            &outputDesc->mDevice,
                                                            address,
                                                            &outputDesc->mLatency,
                                                            outputDesc->mFlags);

            if (status != NO_ERROR) {
                ALOGW("Cannot open output stream for device %08x on hw module %s",
                      outputDesc->mDevice,
                      mHwModules[i]->getName());
            } else {
                outputDesc->mSamplingRate = config.sample_rate;
                outputDesc->mChannelMask = config.channel_mask;
                outputDesc->mFormat = config.format;

                for (size_t k = 0; k  < supportedDevices.size(); k++) {
                    ssize_t index = mAvailableOutputDevices.indexOf(supportedDevices[k]);
                    // give a valid ID to an attached device once confirmed it is reachable
                    if (index >= 0 && !mAvailableOutputDevices[index]->isAttached()) {
                        mAvailableOutputDevices[index]->attach(mHwModules[i]);
                    }
                }
                if (mPrimaryOutput == 0 &&
                        outProfile->getFlags() & AUDIO_OUTPUT_FLAG_PRIMARY) {
                    mPrimaryOutput = outputDesc;
#ifdef MTK_AUDIO    //  Load volume setting from HAL
                    InitSamplerateArray(mPrimaryOutput->mSamplingRate);
#ifdef MTK_NEW_VOL_CONTROL
                    if (mpClientInterface->getCustomAudioVolume(&mGainTable)) {
                        ALOGE("error, load GainTable failed!!");
                        mAudioPolicyVendorControl.setCustomVolumeStatus(false);
                    } else {
                        mAudioPolicyVendorControl.setCustomVolumeStatus(true);
                    }
                    /* XML changed callback process */
                    AppOps* appOps = appOpsGetInstance();
                    if (appOps == NULL) {
                        ALOGE("%s(), Error: AppOps == NULL", __FUNCTION__);
                        continue;
                    }

                    appOps->appHandleRegXmlChangedCb(appOps->appHandleGetInstance(), gainTableXmlChangedCb);
#else
                    mAudioCustVolumeTable.bRev = CUSTOM_VOLUME_REV_1;
                    mAudioCustVolumeTable.bReady = 0;
                    mpClientInterface->getCustomAudioVolume(&mAudioCustVolumeTable);
                    if (mAudioCustVolumeTable.bReady!=0) {
                        ALOGD("mUseCustomVolume true");
                        mAudioPolicyVendorControl.setCustomVolumeStatus(true);
                    } else {
                        ALOGD("mUseCustomVolume false");
                    }
#endif
#endif
                }
                addOutput(output, outputDesc);
                setOutputDevice(outputDesc,
                                outputDesc->mDevice,
                                true,
                                0,
                                NULL,
                                address.string());
            }
        }
        // open input streams needed to access attached devices to validate
        // mAvailableInputDevices list
        for (size_t j = 0; j < mHwModules[i]->mInputProfiles.size(); j++)
        {
            const sp<IOProfile> inProfile = mHwModules[i]->mInputProfiles[j];

            if (!inProfile->hasSupportedDevices()) {
                ALOGW("Input profile contains no device on module %s", mHwModules[i]->getName());
                continue;
            }
            // chose first device present in profile's SupportedDevices also part of
            // inputDeviceTypes
            audio_devices_t profileType = inProfile->getSupportedDeviceForType(inputDeviceTypes);

            if ((profileType & inputDeviceTypes) == 0) {
                continue;
            }
            sp<AudioInputDescriptor> inputDesc =
                    new AudioInputDescriptor(inProfile);

            inputDesc->mDevice = profileType;

            // find the address
            DeviceVector inputDevices = mAvailableInputDevices.getDevicesFromType(profileType);
            //   the inputs vector must be of size 1, but we don't want to crash here
            String8 address = inputDevices.size() > 0 ? inputDevices.itemAt(0)->mAddress
                    : String8("");
            ALOGV("  for input device 0x%x using address %s", profileType, address.string());
            ALOGE_IF(inputDevices.size() == 0, "Input device list is empty!");

            audio_config_t config = AUDIO_CONFIG_INITIALIZER;
            config.sample_rate = inputDesc->mSamplingRate;
            config.channel_mask = inputDesc->mChannelMask;
            config.format = inputDesc->mFormat;
            audio_io_handle_t input = AUDIO_IO_HANDLE_NONE;
            status_t status = mpClientInterface->openInput(inProfile->getModuleHandle(),
                                                           &input,
                                                           &config,
                                                           &inputDesc->mDevice,
                                                           address,
                                                           AUDIO_SOURCE_MIC,
                                                           AUDIO_INPUT_FLAG_NONE);

            if (status == NO_ERROR) {
                const DeviceVector &supportedDevices = inProfile->getSupportedDevices();
                for (size_t k = 0; k  < supportedDevices.size(); k++) {
                    ssize_t index =  mAvailableInputDevices.indexOf(supportedDevices[k]);
                    // give a valid ID to an attached device once confirmed it is reachable
                    if (index >= 0) {
                        sp<DeviceDescriptor> devDesc = mAvailableInputDevices[index];
                        if (!devDesc->isAttached()) {
                            devDesc->attach(mHwModules[i]);
                            devDesc->importAudioPort(inProfile);
                        }
                    }
                }
                mpClientInterface->closeInput(input);
            } else {
                ALOGW("Cannot open input stream for device %08x on hw module %s",
                      inputDesc->mDevice,
                      mHwModules[i]->getName());
            }
        }
    }
#ifdef MTK_AUDIO
        // We set primary output or deepbuffer output as fmradio ref output id
        bool propertySupportDeepBuffer = property_get_bool("audio.deep_buffer.media", false /* default_value */);
        bool configSupportDeepBuffer = false;
        audio_port_handle_t handleOfDeepBuffer = 0;
        audio_port_handle_t handleOfPrimary = 0;
        audio_port_handle_t handleOfFast = 0;
        const size_t SIZE = 32;
        char buffer[SIZE];
        ALOGV("propertySupportDeepBuffer %d", propertySupportDeepBuffer);
        for (size_t i = 0; i < mOutputs.size(); i++) {
            sp<SwAudioOutputDescriptor> desc = mOutputs.valueAt(i);
            if ((desc->mFlags & AUDIO_OUTPUT_FLAG_PRIMARY) != 0) {
                struct audio_patch patch;
                desc->toAudioPortConfig(&patch.sources[0]);
                handleOfPrimary = patch.sources[0].id;
                ALOGD("check outputID [%d] Primary", handleOfPrimary);
            }
            if (propertySupportDeepBuffer) {
                if ((desc->mFlags & AUDIO_OUTPUT_FLAG_DEEP_BUFFER) != 0) {
                    struct audio_patch patch;
                    configSupportDeepBuffer = true;
                    desc->toAudioPortConfig(&patch.sources[0]);
                    handleOfDeepBuffer = patch.sources[0].id;
                    snprintf(buffer, SIZE, "%d", handleOfDeepBuffer);
                    ALOGD("select outputID [%d] DeepBuffer as FMRadio reference", handleOfDeepBuffer);
                    break;
                }
            }
            if ((desc->mFlags & AUDIO_OUTPUT_FLAG_FAST) != 0) {
                struct audio_patch patch;
                desc->toAudioPortConfig(&patch.sources[0]);
                handleOfFast = patch.sources[0].id;
                ALOGD("check outputID [%d] Fast", handleOfFast);
            }
        }
        if (configSupportDeepBuffer == false) {
            snprintf(buffer, SIZE, "%d", handleOfPrimary);
            ALOGD("select ID [%d] Primary as FMRadio reference", handleOfPrimary);
        }
        ALOGD("af.music.outputid = %s", buffer);
        property_set("af.music.outputid", buffer);
#endif
    // make sure all attached devices have been allocated a unique ID
    for (size_t i = 0; i  < mAvailableOutputDevices.size();) {
        if (!mAvailableOutputDevices[i]->isAttached()) {
            ALOGW("Output device %08x unreachable", mAvailableOutputDevices[i]->type());
            mAvailableOutputDevices.remove(mAvailableOutputDevices[i]);
            continue;
        }
        // The device is now validated and can be appended to the available devices of the engine
        mEngine->setDeviceConnectionState(mAvailableOutputDevices[i],
                                          AUDIO_POLICY_DEVICE_STATE_AVAILABLE);
        i++;
    }
    for (size_t i = 0; i  < mAvailableInputDevices.size();) {
        if (!mAvailableInputDevices[i]->isAttached()) {
            ALOGW("Input device %08x unreachable", mAvailableInputDevices[i]->type());
            mAvailableInputDevices.remove(mAvailableInputDevices[i]);
            continue;
        }
        // The device is now validated and can be appended to the available devices of the engine
        mEngine->setDeviceConnectionState(mAvailableInputDevices[i],
                                          AUDIO_POLICY_DEVICE_STATE_AVAILABLE);
        i++;
    }
    // make sure default device is reachable
    if (mDefaultOutputDevice == 0 || mAvailableOutputDevices.indexOf(mDefaultOutputDevice) < 0) {
        ALOGE("Default device %08x is unreachable", mDefaultOutputDevice->type());
    }

    ALOGE_IF((mPrimaryOutput == 0), "Failed to open primary output");

    updateDevicesAndOutputs();

#ifdef AUDIO_POLICY_TEST
    if (mPrimaryOutput != 0) {
        AudioParameter outputCmd = AudioParameter();
        outputCmd.addInt(String8("set_id"), 0);
        mpClientInterface->setParameters(mPrimaryOutput->mIoHandle, outputCmd.toString());

        mTestDevice = AUDIO_DEVICE_OUT_SPEAKER;
        mTestSamplingRate = 44100;
        mTestFormat = AUDIO_FORMAT_PCM_16_BIT;
        mTestChannels =  AUDIO_CHANNEL_OUT_STEREO;
        mTestLatencyMs = 0;
        mCurOutput = 0;
        mDirectOutput = false;
        for (int i = 0; i < NUM_TEST_OUTPUTS; i++) {
            mTestOutputs[i] = 0;
        }

        const size_t SIZE = 256;
        char buffer[SIZE];
        snprintf(buffer, SIZE, "AudioPolicyManagerTest");
        run(buffer, ANDROID_PRIORITY_AUDIO);
    }
#endif //AUDIO_POLICY_TEST
#ifdef MTK_AUDIO
    ALOGD("-%s %d",__FUNCTION__,__LINE__);
#endif

}

AudioPolicyManager::~AudioPolicyManager()
{
#ifdef AUDIO_POLICY_TEST
    exit();
#endif //AUDIO_POLICY_TEST
   for (size_t i = 0; i < mOutputs.size(); i++) {
        mpClientInterface->closeOutput(mOutputs.keyAt(i));
   }
   for (size_t i = 0; i < mInputs.size(); i++) {
        mpClientInterface->closeInput(mInputs.keyAt(i));
   }
   mAvailableOutputDevices.clear();
   mAvailableInputDevices.clear();
   mOutputs.clear();
   mInputs.clear();
   mHwModules.clear();
}

status_t AudioPolicyManager::initCheck()
{
    return hasPrimaryOutput() ? NO_ERROR : NO_INIT;
}

#ifdef AUDIO_POLICY_TEST
bool AudioPolicyManager::threadLoop()
{
    ALOGV("entering threadLoop()");
    while (!exitPending())
    {
        String8 command;
        int valueInt;
        String8 value;

        Mutex::Autolock _l(mLock);
        mWaitWorkCV.waitRelative(mLock, milliseconds(50));

        command = mpClientInterface->getParameters(0, String8("test_cmd_policy"));
        AudioParameter param = AudioParameter(command);

        if (param.getInt(String8("test_cmd_policy"), valueInt) == NO_ERROR &&
            valueInt != 0) {
            ALOGV("Test command %s received", command.string());
            String8 target;
            if (param.get(String8("target"), target) != NO_ERROR) {
                target = "Manager";
            }
            if (param.getInt(String8("test_cmd_policy_output"), valueInt) == NO_ERROR) {
                param.remove(String8("test_cmd_policy_output"));
                mCurOutput = valueInt;
            }
            if (param.get(String8("test_cmd_policy_direct"), value) == NO_ERROR) {
                param.remove(String8("test_cmd_policy_direct"));
                if (value == "false") {
                    mDirectOutput = false;
                } else if (value == "true") {
                    mDirectOutput = true;
                }
            }
            if (param.getInt(String8("test_cmd_policy_input"), valueInt) == NO_ERROR) {
                param.remove(String8("test_cmd_policy_input"));
                mTestInput = valueInt;
            }

            if (param.get(String8("test_cmd_policy_format"), value) == NO_ERROR) {
                param.remove(String8("test_cmd_policy_format"));
                int format = AUDIO_FORMAT_INVALID;
                if (value == "PCM 16 bits") {
                    format = AUDIO_FORMAT_PCM_16_BIT;
                } else if (value == "PCM 8 bits") {
                    format = AUDIO_FORMAT_PCM_8_BIT;
                } else if (value == "Compressed MP3") {
                    format = AUDIO_FORMAT_MP3;
                }
                if (format != AUDIO_FORMAT_INVALID) {
                    if (target == "Manager") {
                        mTestFormat = format;
                    } else if (mTestOutputs[mCurOutput] != 0) {
                        AudioParameter outputParam = AudioParameter();
                        outputParam.addInt(String8("format"), format);
                        mpClientInterface->setParameters(mTestOutputs[mCurOutput], outputParam.toString());
                    }
                }
            }
            if (param.get(String8("test_cmd_policy_channels"), value) == NO_ERROR) {
                param.remove(String8("test_cmd_policy_channels"));
                int channels = 0;

                if (value == "Channels Stereo") {
                    channels =  AUDIO_CHANNEL_OUT_STEREO;
                } else if (value == "Channels Mono") {
                    channels =  AUDIO_CHANNEL_OUT_MONO;
                }
                if (channels != 0) {
                    if (target == "Manager") {
                        mTestChannels = channels;
                    } else if (mTestOutputs[mCurOutput] != 0) {
                        AudioParameter outputParam = AudioParameter();
                        outputParam.addInt(String8("channels"), channels);
                        mpClientInterface->setParameters(mTestOutputs[mCurOutput], outputParam.toString());
                    }
                }
            }
            if (param.getInt(String8("test_cmd_policy_sampleRate"), valueInt) == NO_ERROR) {
                param.remove(String8("test_cmd_policy_sampleRate"));
                if (valueInt >= 0 && valueInt <= 96000) {
                    int samplingRate = valueInt;
                    if (target == "Manager") {
                        mTestSamplingRate = samplingRate;
                    } else if (mTestOutputs[mCurOutput] != 0) {
                        AudioParameter outputParam = AudioParameter();
                        outputParam.addInt(String8("sampling_rate"), samplingRate);
                        mpClientInterface->setParameters(mTestOutputs[mCurOutput], outputParam.toString());
                    }
                }
            }

            if (param.get(String8("test_cmd_policy_reopen"), value) == NO_ERROR) {
                param.remove(String8("test_cmd_policy_reopen"));

                mpClientInterface->closeOutput(mpClientInterface->closeOutput(mPrimaryOutput););

                audio_module_handle_t moduleHandle = mPrimaryOutput->getModuleHandle();

                removeOutput(mPrimaryOutput->mIoHandle);
                sp<SwAudioOutputDescriptor> outputDesc = new AudioOutputDescriptor(NULL,
                                                                               mpClientInterface);
                outputDesc->mDevice = AUDIO_DEVICE_OUT_SPEAKER;
                audio_config_t config = AUDIO_CONFIG_INITIALIZER;
                config.sample_rate = outputDesc->mSamplingRate;
                config.channel_mask = outputDesc->mChannelMask;
                config.format = outputDesc->mFormat;
                audio_io_handle_t handle;
                status_t status = mpClientInterface->openOutput(moduleHandle,
                                                                &handle,
                                                                &config,
                                                                &outputDesc->mDevice,
                                                                String8(""),
                                                                &outputDesc->mLatency,
                                                                outputDesc->mFlags);
                if (status != NO_ERROR) {
                    ALOGE("Failed to reopen hardware output stream, "
                        "samplingRate: %d, format %d, channels %d",
                        outputDesc->mSamplingRate, outputDesc->mFormat, outputDesc->mChannelMask);
                } else {
                    outputDesc->mSamplingRate = config.sample_rate;
                    outputDesc->mChannelMask = config.channel_mask;
                    outputDesc->mFormat = config.format;
                    mPrimaryOutput = outputDesc;
                    AudioParameter outputCmd = AudioParameter();
                    outputCmd.addInt(String8("set_id"), 0);
                    mpClientInterface->setParameters(handle, outputCmd.toString());
                    addOutput(handle, outputDesc);
                }
            }


            mpClientInterface->setParameters(0, String8("test_cmd_policy="));
        }
    }
    return false;
}

void AudioPolicyManager::exit()
{
    {
        AutoMutex _l(mLock);
        requestExit();
        mWaitWorkCV.signal();
    }
    requestExitAndWait();
}

int AudioPolicyManager::testOutputIndex(audio_io_handle_t output)
{
    for (int i = 0; i < NUM_TEST_OUTPUTS; i++) {
        if (output == mTestOutputs[i]) return i;
    }
    return 0;
}
#endif //AUDIO_POLICY_TEST

// ---

void AudioPolicyManager::addOutput(audio_io_handle_t output, sp<SwAudioOutputDescriptor> outputDesc)
{
    outputDesc->setIoHandle(output);
    mOutputs.add(output, outputDesc);
    updateMono(output); // update mono status when adding to output list
    nextAudioPortGeneration();
}

void AudioPolicyManager::removeOutput(audio_io_handle_t output)
{
    mOutputs.removeItem(output);
}

void AudioPolicyManager::addInput(audio_io_handle_t input, sp<AudioInputDescriptor> inputDesc)
{
    inputDesc->setIoHandle(input);
    mInputs.add(input, inputDesc);
    nextAudioPortGeneration();
}

void AudioPolicyManager::findIoHandlesByAddress(sp<SwAudioOutputDescriptor> desc /*in*/,
        const audio_devices_t device /*in*/,
        const String8 address /*in*/,
        SortedVector<audio_io_handle_t>& outputs /*out*/) {
    sp<DeviceDescriptor> devDesc =
        desc->mProfile->getSupportedDeviceByAddress(device, address);
    if (devDesc != 0) {
        ALOGV("findIoHandlesByAddress(): adding opened output %d on same address %s",
              desc->mIoHandle, address.string());
        outputs.add(desc->mIoHandle);
    }
}

status_t AudioPolicyManager::checkOutputsForDevice(const sp<DeviceDescriptor> devDesc,
                                                   audio_policy_dev_state_t state,
                                                   SortedVector<audio_io_handle_t>& outputs,
                                                   const String8 address)
{
    audio_devices_t device = devDesc->type();
    sp<SwAudioOutputDescriptor> desc;

    if (audio_device_is_digital(device)) {
        // erase all current sample rates, formats and channel masks
        devDesc->clearAudioProfiles();
    }

    if (state == AUDIO_POLICY_DEVICE_STATE_AVAILABLE) {
        // first list already open outputs that can be routed to this device
        for (size_t i = 0; i < mOutputs.size(); i++) {
            desc = mOutputs.valueAt(i);
            if (!desc->isDuplicated() && (desc->supportedDevices() & device)) {
                if (!device_distinguishes_on_address(device)) {
                    ALOGV("checkOutputsForDevice(): adding opened output %d", mOutputs.keyAt(i));
                    outputs.add(mOutputs.keyAt(i));
                } else {
                    ALOGV("  checking address match due to device 0x%x", device);
                    findIoHandlesByAddress(desc, device, address, outputs);
                }
            }
        }
        // then look for output profiles that can be routed to this device
        SortedVector< sp<IOProfile> > profiles;
        for (size_t i = 0; i < mHwModules.size(); i++)
        {
            if (mHwModules[i]->mHandle == 0) {
                continue;
            }
            for (size_t j = 0; j < mHwModules[i]->mOutputProfiles.size(); j++)
            {
                sp<IOProfile> profile = mHwModules[i]->mOutputProfiles[j];
                if (profile->supportDevice(device)) {
                    if (!device_distinguishes_on_address(device) ||
                            profile->supportDeviceAddress(address)) {
                        profiles.add(profile);
                        ALOGV("checkOutputsForDevice(): adding profile %zu from module %zu", j, i);
                    }
                }
            }
        }

        ALOGV("  found %zu profiles, %zu outputs", profiles.size(), outputs.size());

        if (profiles.isEmpty() && outputs.isEmpty()) {
            ALOGW("checkOutputsForDevice(): No output available for device %04x", device);
            return BAD_VALUE;
        }

        // open outputs for matching profiles if needed. Direct outputs are also opened to
        // query for dynamic parameters and will be closed later by setDeviceConnectionState()
        for (ssize_t profile_index = 0; profile_index < (ssize_t)profiles.size(); profile_index++) {
            sp<IOProfile> profile = profiles[profile_index];

            // nothing to do if one output is already opened for this profile
            size_t j;
            for (j = 0; j < outputs.size(); j++) {
                desc = mOutputs.valueFor(outputs.itemAt(j));
                if (!desc->isDuplicated() && desc->mProfile == profile) {
                    // matching profile: save the sample rates, format and channel masks supported
                    // by the profile in our device descriptor
                    if (audio_device_is_digital(device)) {
                        devDesc->importAudioPort(profile);
                    }
                    break;
                }
            }
            if (j != outputs.size()) {
                continue;
            }

            ALOGV("opening output for device %08x with params %s profile %p",
                                                      device, address.string(), profile.get());
            desc = new SwAudioOutputDescriptor(profile, mpClientInterface);
            desc->mDevice = device;
            audio_config_t config = AUDIO_CONFIG_INITIALIZER;
            config.sample_rate = desc->mSamplingRate;
            config.channel_mask = desc->mChannelMask;
            config.format = desc->mFormat;
            config.offload_info.sample_rate = desc->mSamplingRate;
            config.offload_info.channel_mask = desc->mChannelMask;
            config.offload_info.format = desc->mFormat;
            audio_io_handle_t output = AUDIO_IO_HANDLE_NONE;
            status_t status = mpClientInterface->openOutput(profile->getModuleHandle(),
                                                            &output,
                                                            &config,
                                                            &desc->mDevice,
                                                            address,
                                                            &desc->mLatency,
                                                            desc->mFlags);
            if (status == NO_ERROR) {
                desc->mSamplingRate = config.sample_rate;
                desc->mChannelMask = config.channel_mask;
                desc->mFormat = config.format;

                // Here is where the out_set_parameters() for card & device gets called
                if (!address.isEmpty()) {
                    char *param = audio_device_address_to_parameter(device, address);
                    mpClientInterface->setParameters(output, String8(param));
                    free(param);
                }
                updateAudioProfiles(device, output, profile->getAudioProfiles());
                if (!profile->hasValidAudioProfile()) {
                    ALOGW("checkOutputsForDevice() missing param");
                    mpClientInterface->closeOutput(output);
                    output = AUDIO_IO_HANDLE_NONE;
                } else if (profile->hasDynamicAudioProfile()) {
                    mpClientInterface->closeOutput(output);
                    output = AUDIO_IO_HANDLE_NONE;
                    profile->pickAudioProfile(config.sample_rate, config.channel_mask, config.format);
                    config.offload_info.sample_rate = config.sample_rate;
                    config.offload_info.channel_mask = config.channel_mask;
                    config.offload_info.format = config.format;
                    status = mpClientInterface->openOutput(profile->getModuleHandle(),
                                                           &output,
                                                           &config,
                                                           &desc->mDevice,
                                                           address,
                                                           &desc->mLatency,
                                                           desc->mFlags);
                    if (status == NO_ERROR) {
                        desc->mSamplingRate = config.sample_rate;
                        desc->mChannelMask = config.channel_mask;
                        desc->mFormat = config.format;
                    } else {
                        output = AUDIO_IO_HANDLE_NONE;
                    }
                }

                if (output != AUDIO_IO_HANDLE_NONE) {
                    addOutput(output, desc);
                    if (device_distinguishes_on_address(device) && address != "0") {
                        sp<AudioPolicyMix> policyMix;
                        if (mPolicyMixes.getAudioPolicyMix(address, policyMix) != NO_ERROR) {
                            ALOGE("checkOutputsForDevice() cannot find policy for address %s",
                                  address.string());
                        }
                        policyMix->setOutput(desc);
                        desc->mPolicyMix = policyMix->getMix();

                    } else if (((desc->mFlags & AUDIO_OUTPUT_FLAG_DIRECT) == 0) &&
                                    hasPrimaryOutput()) {
                        // no duplicated output for direct outputs and
                        // outputs used by dynamic policy mixes
                        audio_io_handle_t duplicatedOutput = AUDIO_IO_HANDLE_NONE;

                        // set initial stream volume for device
                        applyStreamVolumes(desc, device, 0, true);

                        //TODO: configure audio effect output stage here

                        // open a duplicating output thread for the new output and the primary output
                        duplicatedOutput =
                                mpClientInterface->openDuplicateOutput(output,
                                                                       mPrimaryOutput->mIoHandle);
                        if (duplicatedOutput != AUDIO_IO_HANDLE_NONE) {
                            // add duplicated output descriptor
                            sp<SwAudioOutputDescriptor> dupOutputDesc =
                                    new SwAudioOutputDescriptor(NULL, mpClientInterface);
                            dupOutputDesc->mOutput1 = mPrimaryOutput;
                            dupOutputDesc->mOutput2 = desc;
                            dupOutputDesc->mSamplingRate = desc->mSamplingRate;
                            dupOutputDesc->mFormat = desc->mFormat;
                            dupOutputDesc->mChannelMask = desc->mChannelMask;
                            dupOutputDesc->mLatency = desc->mLatency;
                            addOutput(duplicatedOutput, dupOutputDesc);
                            applyStreamVolumes(dupOutputDesc, device, 0, true);
                        } else {
                            ALOGW("checkOutputsForDevice() could not open dup output for %d and %d",
                                    mPrimaryOutput->mIoHandle, output);
                            mpClientInterface->closeOutput(output);
                            removeOutput(output);
                            nextAudioPortGeneration();
                            output = AUDIO_IO_HANDLE_NONE;
                        }
                    }
                }
            } else {
                output = AUDIO_IO_HANDLE_NONE;
#ifdef MTK_AUDIO
                ALOGE("Error %s#Line%d output fail",__FUNCTION__,__LINE__);
#endif
            }
            if (output == AUDIO_IO_HANDLE_NONE) {
                ALOGW("checkOutputsForDevice() could not open output for device %x", device);
                profiles.removeAt(profile_index);
                profile_index--;
            } else {
                outputs.add(output);
                // Load digital format info only for digital devices
                if (audio_device_is_digital(device)) {
                    devDesc->importAudioPort(profile);
                }

                if (device_distinguishes_on_address(device)) {
                    ALOGV("checkOutputsForDevice(): setOutputDevice(dev=0x%x, addr=%s)",
                            device, address.string());
                    setOutputDevice(desc, device, true/*force*/, 0/*delay*/,
                            NULL/*patch handle*/, address.string());
                }
                ALOGV("checkOutputsForDevice(): adding output %d", output);
            }
        }

        if (profiles.isEmpty()) {
            ALOGW("checkOutputsForDevice(): No output available for device %04x", device);
            return BAD_VALUE;
        }
    } else { // Disconnect
        // check if one opened output is not needed any more after disconnecting one device
        for (size_t i = 0; i < mOutputs.size(); i++) {
            desc = mOutputs.valueAt(i);
            if (!desc->isDuplicated()) {
                // exact match on device
                if (device_distinguishes_on_address(device) &&
                        (desc->supportedDevices() == device)) {
                    findIoHandlesByAddress(desc, device, address, outputs);
                } else if (!(desc->supportedDevices() & mAvailableOutputDevices.types())) {
                    ALOGV("checkOutputsForDevice(): disconnecting adding output %d",
                            mOutputs.keyAt(i));
                    outputs.add(mOutputs.keyAt(i));
                }
            }
        }
        // Clear any profiles associated with the disconnected device.
        for (size_t i = 0; i < mHwModules.size(); i++)
        {
            if (mHwModules[i]->mHandle == 0) {
                continue;
            }
            for (size_t j = 0; j < mHwModules[i]->mOutputProfiles.size(); j++)
            {
                sp<IOProfile> profile = mHwModules[i]->mOutputProfiles[j];
                if (profile->supportDevice(device)) {
                    ALOGV("checkOutputsForDevice(): "
                            "clearing direct output profile %zu on module %zu", j, i);
                    profile->clearAudioProfiles();
                }
            }
        }
    }
    return NO_ERROR;
}

status_t AudioPolicyManager::checkInputsForDevice(const sp<DeviceDescriptor> devDesc,
                                                  audio_policy_dev_state_t state,
                                                  SortedVector<audio_io_handle_t>& inputs,
                                                  const String8 address)
{
    audio_devices_t device = devDesc->type();
    sp<AudioInputDescriptor> desc;

    if (audio_device_is_digital(device)) {
        // erase all current sample rates, formats and channel masks
        devDesc->clearAudioProfiles();
    }

    if (state == AUDIO_POLICY_DEVICE_STATE_AVAILABLE) {
        // first list already open inputs that can be routed to this device
        for (size_t input_index = 0; input_index < mInputs.size(); input_index++) {
            desc = mInputs.valueAt(input_index);
            if (desc->mProfile->supportDevice(device)) {
                ALOGV("checkInputsForDevice(): adding opened input %d", mInputs.keyAt(input_index));
               inputs.add(mInputs.keyAt(input_index));
            }
        }

        // then look for input profiles that can be routed to this device
        SortedVector< sp<IOProfile> > profiles;
        for (size_t module_idx = 0; module_idx < mHwModules.size(); module_idx++)
        {
            if (mHwModules[module_idx]->mHandle == 0) {
                continue;
            }
            for (size_t profile_index = 0;
                 profile_index < mHwModules[module_idx]->mInputProfiles.size();
                 profile_index++)
            {
                sp<IOProfile> profile = mHwModules[module_idx]->mInputProfiles[profile_index];

                if (profile->supportDevice(device)) {
                    if (!device_distinguishes_on_address(device) ||
                            profile->supportDeviceAddress(address)) {
                        profiles.add(profile);
                        ALOGV("checkInputsForDevice(): adding profile %zu from module %zu",
                              profile_index, module_idx);
                    }
                }
            }
        }

        if (profiles.isEmpty() && inputs.isEmpty()) {
            ALOGW("checkInputsForDevice(): No input available for device 0x%X", device);
            return BAD_VALUE;
        }

        // open inputs for matching profiles if needed. Direct inputs are also opened to
        // query for dynamic parameters and will be closed later by setDeviceConnectionState()
        for (ssize_t profile_index = 0; profile_index < (ssize_t)profiles.size(); profile_index++) {

            sp<IOProfile> profile = profiles[profile_index];
            // nothing to do if one input is already opened for this profile
            size_t input_index;
            for (input_index = 0; input_index < mInputs.size(); input_index++) {
                desc = mInputs.valueAt(input_index);
                if (desc->mProfile == profile) {
                    if (audio_device_is_digital(device)) {
                        devDesc->importAudioPort(profile);
                    }
                    break;
                }
            }
            if (input_index != mInputs.size()) {
                continue;
            }

            ALOGV("opening input for device 0x%X with params %s", device, address.string());
            desc = new AudioInputDescriptor(profile);
            desc->mDevice = device;
            audio_config_t config = AUDIO_CONFIG_INITIALIZER;
            config.sample_rate = desc->mSamplingRate;
            config.channel_mask = desc->mChannelMask;
            config.format = desc->mFormat;
            audio_io_handle_t input = AUDIO_IO_HANDLE_NONE;
            status_t status = mpClientInterface->openInput(profile->getModuleHandle(),
                                                           &input,
                                                           &config,
                                                           &desc->mDevice,
                                                           address,
                                                           AUDIO_SOURCE_MIC,
                                                           AUDIO_INPUT_FLAG_NONE /*FIXME*/);

            if (status == NO_ERROR) {
                desc->mSamplingRate = config.sample_rate;
                desc->mChannelMask = config.channel_mask;
                desc->mFormat = config.format;

                if (!address.isEmpty()) {
                    char *param = audio_device_address_to_parameter(device, address);
                    mpClientInterface->setParameters(input, String8(param));
                    free(param);
                }
                updateAudioProfiles(device, input, profile->getAudioProfiles());
                if (!profile->hasValidAudioProfile()) {
                    ALOGW("checkInputsForDevice() direct input missing param");
                    mpClientInterface->closeInput(input);
                    input = AUDIO_IO_HANDLE_NONE;
                }

                if (input != 0) {
#ifdef MTK_AUDIO
                    ALOGD("input %d %s %d",input,__FUNCTION__,__LINE__);
#endif
                    addInput(input, desc);
                }
            } // endif input != 0

            if (input == AUDIO_IO_HANDLE_NONE) {
                ALOGW("checkInputsForDevice() could not open input for device 0x%X", device);
                profiles.removeAt(profile_index);
                profile_index--;
            } else {
                inputs.add(input);
                if (audio_device_is_digital(device)) {
                    devDesc->importAudioPort(profile);
                }
                ALOGV("checkInputsForDevice(): adding input %d", input);
            }
        } // end scan profiles

        if (profiles.isEmpty()) {
            ALOGW("checkInputsForDevice(): No input available for device 0x%X", device);
            return BAD_VALUE;
        }
    } else {
        // Disconnect
        // check if one opened input is not needed any more after disconnecting one device
        for (size_t input_index = 0; input_index < mInputs.size(); input_index++) {
            desc = mInputs.valueAt(input_index);
            if (!(desc->mProfile->supportDevice(mAvailableInputDevices.types()))) {
                ALOGV("checkInputsForDevice(): disconnecting adding input %d",
                      mInputs.keyAt(input_index));
                inputs.add(mInputs.keyAt(input_index));
            }
        }
        // Clear any profiles associated with the disconnected device.
        for (size_t module_index = 0; module_index < mHwModules.size(); module_index++) {
            if (mHwModules[module_index]->mHandle == 0) {
                continue;
            }
            for (size_t profile_index = 0;
                 profile_index < mHwModules[module_index]->mInputProfiles.size();
                 profile_index++) {
                sp<IOProfile> profile = mHwModules[module_index]->mInputProfiles[profile_index];
                if (profile->supportDevice(device)) {
                    ALOGV("checkInputsForDevice(): clearing direct input profile %zu on module %zu",
                          profile_index, module_index);
                    profile->clearAudioProfiles();
                }
            }
        }
    } // end disconnect

    return NO_ERROR;
}


void AudioPolicyManager::closeOutput(audio_io_handle_t output)
{
    ALOGV("closeOutput(%d)", output);

    sp<SwAudioOutputDescriptor> outputDesc = mOutputs.valueFor(output);
    if (outputDesc == NULL) {
        ALOGW("closeOutput() unknown output %d", output);
        return;
    }
    mPolicyMixes.closeOutput(outputDesc);

    // look for duplicated outputs connected to the output being removed.
    for (size_t i = 0; i < mOutputs.size(); i++) {
        sp<SwAudioOutputDescriptor> dupOutputDesc = mOutputs.valueAt(i);
        if (dupOutputDesc->isDuplicated() &&
                (dupOutputDesc->mOutput1 == outputDesc ||
                dupOutputDesc->mOutput2 == outputDesc)) {
            sp<AudioOutputDescriptor> outputDesc2;
            if (dupOutputDesc->mOutput1 == outputDesc) {
                outputDesc2 = dupOutputDesc->mOutput2;
            } else {
                outputDesc2 = dupOutputDesc->mOutput1;
            }
            // As all active tracks on duplicated output will be deleted,
            // and as they were also referenced on the other output, the reference
            // count for their stream type must be adjusted accordingly on
            // the other output.
            for (int j = 0; j < AUDIO_STREAM_CNT; j++) {
                int refCount = dupOutputDesc->mRefCount[j];
                outputDesc2->changeRefCount((audio_stream_type_t)j,-refCount);
            }
            audio_io_handle_t duplicatedOutput = mOutputs.keyAt(i);
            ALOGV("closeOutput() closing also duplicated output %d", duplicatedOutput);

            mpClientInterface->closeOutput(duplicatedOutput);
            removeOutput(duplicatedOutput);
        }
    }

    nextAudioPortGeneration();

    ssize_t index = mAudioPatches.indexOfKey(outputDesc->getPatchHandle());
    if (index >= 0) {
        sp<AudioPatch> patchDesc = mAudioPatches.valueAt(index);
        status_t status = mpClientInterface->releaseAudioPatch(patchDesc->mAfPatchHandle, 0);
        mAudioPatches.removeItemsAt(index);
        mpClientInterface->onAudioPatchListUpdate();
    }

    AudioParameter param;
    param.add(String8("closing"), String8("true"));
    mpClientInterface->setParameters(output, param.toString());

    mpClientInterface->closeOutput(output);
    removeOutput(output);
    mPreviousOutputs = mOutputs;
}

void AudioPolicyManager::closeInput(audio_io_handle_t input)
{
    ALOGV("closeInput(%d)", input);

    sp<AudioInputDescriptor> inputDesc = mInputs.valueFor(input);
    if (inputDesc == NULL) {
        ALOGW("closeInput() unknown input %d", input);
        return;
    }

    nextAudioPortGeneration();

    ssize_t index = mAudioPatches.indexOfKey(inputDesc->getPatchHandle());
    if (index >= 0) {
        sp<AudioPatch> patchDesc = mAudioPatches.valueAt(index);
        status_t status = mpClientInterface->releaseAudioPatch(patchDesc->mAfPatchHandle, 0);
        mAudioPatches.removeItemsAt(index);
        mpClientInterface->onAudioPatchListUpdate();
    }

    mpClientInterface->closeInput(input);
    mInputs.removeItem(input);
}

SortedVector<audio_io_handle_t> AudioPolicyManager::getOutputsForDevice(
                                                                audio_devices_t device,
                                                                SwAudioOutputCollection openOutputs)
{
    SortedVector<audio_io_handle_t> outputs;

#ifdef MTK_AUDIO
    MTK_ALOGV("getOutputsForDevice() device %04x", device);
#else
    ALOGVV("getOutputsForDevice() device %04x", device);
#endif
    for (size_t i = 0; i < openOutputs.size(); i++) {
#ifdef MTK_AUDIO
        MTK_ALOGV("output %zu isDuplicated=%d device=%04x",
                i, openOutputs.valueAt(i)->isDuplicated(),
                openOutputs.valueAt(i)->supportedDevices());
#else
        ALOGVV("output %d isDuplicated=%d device=%04x",
                i, openOutputs.valueAt(i)->isDuplicated(),
                openOutputs.valueAt(i)->supportedDevices());
#endif
        if ((device & openOutputs.valueAt(i)->supportedDevices()) == device) {
#ifdef MTK_AUDIO   // ALPS02401994
            // if device is speaker , fast output don't support VOIP
            if ((AUDIO_MODE_IN_COMMUNICATION == mEngine->getPhoneState()) &&
               (device & AUDIO_DEVICE_OUT_SPEAKER)) {
                if((openOutputs.valueAt(i)->mProfile != 0) &&
                   (openOutputs.valueAt(i)->mProfile->getFlags() & AUDIO_OUTPUT_FLAG_FAST) &&
                   !(openOutputs.valueAt(i)->mProfile->getFlags() & AUDIO_OUTPUT_FLAG_PRIMARY)) {
                   MTK_ALOGV("device is speaker and fast output don't support VOIP, outout flags 0x%x,phone state 0x%x, device 0x%x",
                        openOutputs.valueAt(i)->mProfile->getFlags(), mEngine->getPhoneState(), device);
                    continue;
                }
            }
#ifdef MTK_HDMI_MULTI_CHANNEL_SUPPORT
            // if HDMI only support 2ch, do not use hdmi_multi_channel
            if((device & AUDIO_DEVICE_OUT_AUX_DIGITAL) &&
               (openOutputs.valueAt(i)->mProfile->getAudioProfiles().getChannels() & AUDIO_CHANNEL_OUT_STEREO) != AUDIO_CHANNEL_OUT_STEREO &&
               mAudioPolicyVendorControl.getHDMI_ChannelCount() == 2){
               continue;
            }
#endif
#endif
#ifdef MTK_AUDIO
            MTK_ALOGV("getOutputsForDevice() found output %d", openOutputs.keyAt(i));
#else
            ALOGVV("getOutputsForDevice() found output %d", openOutputs.keyAt(i));
#endif
            outputs.add(openOutputs.keyAt(i));
        }
    }
    return outputs;
}

bool AudioPolicyManager::vectorsEqual(SortedVector<audio_io_handle_t>& outputs1,
                                      SortedVector<audio_io_handle_t>& outputs2)
{
#ifdef MTK_AUDIO    //  If someone is empty, it won't re-routing
    if ((outputs1.isEmpty() || outputs2.isEmpty())) {
        return true;
    }
#endif
    if (outputs1.size() != outputs2.size()) {
        return false;
    }
    for (size_t i = 0; i < outputs1.size(); i++) {
        if (outputs1[i] != outputs2[i]) {
            return false;
        }
    }
    return true;
}

void AudioPolicyManager::checkOutputForStrategy(routing_strategy strategy)
{
    audio_devices_t oldDevice = getDeviceForStrategy(strategy, true /*fromCache*/);
    audio_devices_t newDevice = getDeviceForStrategy(strategy, false /*fromCache*/);
    SortedVector<audio_io_handle_t> srcOutputs = getOutputsForDevice(oldDevice, mPreviousOutputs);
    SortedVector<audio_io_handle_t> dstOutputs = getOutputsForDevice(newDevice, mOutputs);

#ifdef MTK_AUDIO
#if 0
        ALOGD("checkOutputForStrategy()");
        ALOGD("oldDevice [0x%d] newDevice [0x%d]",oldDevice,newDevice);
        ALOGD("srcOutputs size [%d]",srcOutputs.size());
        ALOGD("dstOutputs size [%d]",dstOutputs.size());
        for (size_t i = 0; i < srcOutputs.size(); i++) {
            ALOGD("srcOutputs[%d] [%d]",i,srcOutputs[i]);
        }
        for (size_t i = 0; i < dstOutputs.size(); i++) {
            ALOGD("dstOutputs[%d] [%d]",i,dstOutputs[i]);
        }
#endif
#endif
    // also take into account external policy-related changes: add all outputs which are
    // associated with policies in the "before" and "after" output vectors
#ifndef MTK_AUDIO
    ALOGVV("checkOutputForStrategy(): policy related outputs");
#endif
    for (size_t i = 0 ; i < mPreviousOutputs.size() ; i++) {
        const sp<SwAudioOutputDescriptor> desc = mPreviousOutputs.valueAt(i);
        if (desc != 0 && desc->mPolicyMix != NULL) {
            srcOutputs.add(desc->mIoHandle);
            ALOGVV(" previous outputs: adding %d", desc->mIoHandle);
        }
    }
    for (size_t i = 0 ; i < mOutputs.size() ; i++) {
        const sp<SwAudioOutputDescriptor> desc = mOutputs.valueAt(i);
        if (desc != 0 && desc->mPolicyMix != NULL) {
            dstOutputs.add(desc->mIoHandle);
            ALOGVV(" new outputs: adding %d", desc->mIoHandle);
        }
    }

    if (!vectorsEqual(srcOutputs,dstOutputs)) {
        ALOGV("checkOutputForStrategy() strategy %d, moving from output %d to output %d",
              strategy, srcOutputs[0], dstOutputs[0]);
        // mute strategy while moving tracks from one output to another
        for (size_t i = 0; i < srcOutputs.size(); i++) {
            sp<SwAudioOutputDescriptor> desc = mOutputs.valueFor(srcOutputs[i]);
            if (isStrategyActive(desc, strategy)) {
                setStrategyMute(strategy, true, desc);
                setStrategyMute(strategy, false, desc, MUTE_TIME_MS, newDevice);
            }
            sp<AudioSourceDescriptor> source =
                    getSourceForStrategyOnOutput(srcOutputs[i], strategy);
            if (source != 0){
                connectAudioSource(source);
            }
        }

        // Move effects associated to this strategy from previous output to new output
        if (strategy == STRATEGY_MEDIA) {
            audio_io_handle_t fxOutput = selectOutputForEffects(dstOutputs);
            SortedVector<audio_io_handle_t> moved;
#ifdef MTK_AUDIO
            int preSessionId = 0;
#endif
            for (size_t i = 0; i < mEffects.size(); i++) {
                sp<EffectDescriptor> effectDesc = mEffects.valueAt(i);
#ifdef MTK_AUDIO
                if (effectDesc->mSession != AUDIO_SESSION_OUTPUT_STAGE &&
                        effectDesc->mIo != fxOutput) {
                    if (moved.indexOf(effectDesc->mIo) < 0 || effectDesc->mSession != preSessionId) {
                        ALOGD("checkOutputForStrategy() moving session:%d,effect:%d to output:%d",
                              effectDesc->mSession, mEffects.keyAt(i), fxOutput);
#ifdef DOLBY_DAP
                        status_t status = mpClientInterface->moveEffects(AUDIO_SESSION_OUTPUT_MIX, effectDesc->mIo,
                                                       fxOutput);
                        if (status != NO_ERROR) {
                            ALOGW("%s moveEffects from %d to %d failed", __FUNCTION__, effectDesc->mIo, fxOutput);
                            continue;
                        }
#else // DOLBY_END
                        mpClientInterface->moveEffects((audio_session_t)effectDesc->mSession, effectDesc->mIo,
                                                       fxOutput);
#endif // LINE_ADDED_BY_DOLBY
                        moved.add(effectDesc->mIo);
                        preSessionId = effectDesc->mSession;
                    }
                    effectDesc->mIo = fxOutput;
                }
#else

                if (effectDesc->mSession == AUDIO_SESSION_OUTPUT_MIX &&
                        effectDesc->mIo != fxOutput) {
                    if (moved.indexOf(effectDesc->mIo) < 0) {
                        ALOGV("checkOutputForStrategy() moving effect %d to output %d",
                              mEffects.keyAt(i), fxOutput);
                        mpClientInterface->moveEffects(AUDIO_SESSION_OUTPUT_MIX, effectDesc->mIo,
                                                       fxOutput);
                        moved.add(effectDesc->mIo);
                    }
                    effectDesc->mIo = fxOutput;
                }
#endif
            }
        }
        // Move tracks associated to this strategy from previous output to new output
        for (int i = 0; i < AUDIO_STREAM_FOR_POLICY_CNT; i++) {
            if (getStrategy((audio_stream_type_t)i) == strategy) {
                mpClientInterface->invalidateStream((audio_stream_type_t)i);
            }
        }
    }
}

void AudioPolicyManager::checkOutputForAllStrategies()
{
#ifdef MTK_AUDIO
    ALOGV("checkOutputForAllStrategies");
#endif
    if (mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_SYSTEM) == AUDIO_POLICY_FORCE_SYSTEM_ENFORCED)
        checkOutputForStrategy(STRATEGY_ENFORCED_AUDIBLE);
    checkOutputForStrategy(STRATEGY_PHONE);
    if (mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_SYSTEM) != AUDIO_POLICY_FORCE_SYSTEM_ENFORCED)
        checkOutputForStrategy(STRATEGY_ENFORCED_AUDIBLE);
    checkOutputForStrategy(STRATEGY_SONIFICATION);
    checkOutputForStrategy(STRATEGY_SONIFICATION_RESPECTFUL);
    checkOutputForStrategy(STRATEGY_ACCESSIBILITY);
    checkOutputForStrategy(STRATEGY_MEDIA);
    checkOutputForStrategy(STRATEGY_DTMF);
    checkOutputForStrategy(STRATEGY_REROUTING);
}

void AudioPolicyManager::checkA2dpSuspend()
{
    audio_io_handle_t a2dpOutput = mOutputs.getA2dpOutput();
    if (a2dpOutput == 0) {
        mA2dpSuspended = false;
        return;
    }

    bool isScoConnected =
            ((mAvailableInputDevices.types() & AUDIO_DEVICE_IN_BLUETOOTH_SCO_HEADSET &
                    ~AUDIO_DEVICE_BIT_IN) != 0) ||
            ((mAvailableOutputDevices.types() & AUDIO_DEVICE_OUT_ALL_SCO) != 0);
    // suspend A2DP output if:
    //      (NOT already suspended) &&
    //      ((SCO device is connected &&
    //       (forced usage for communication || for record is SCO))) ||
    //      (phone state is ringing || in call)
    //
    // restore A2DP output if:
    //      (Already suspended) &&
    //      ((SCO device is NOT connected ||
    //       (forced usage NOT for communication && NOT for record is SCO))) &&
    //      (phone state is NOT ringing && NOT in call)
    //
    if (mA2dpSuspended) {
        if ((!isScoConnected ||
             ((mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_COMMUNICATION) != AUDIO_POLICY_FORCE_BT_SCO) &&
              (mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_RECORD) != AUDIO_POLICY_FORCE_BT_SCO))) &&
             ((mEngine->getPhoneState() != AUDIO_MODE_IN_CALL) &&
#ifdef MTK_AUDIO
              (mEngine->getPhoneState() != AUDIO_MODE_IN_CALL_2) &&
              (mEngine->getPhoneState() != AUDIO_MODE_IN_CALL_EXTERNAL) &&
#endif
              (mEngine->getPhoneState() != AUDIO_MODE_RINGTONE))
#ifdef MTK_AUDIO
              && (!mAudioPolicyVendorControl.getA2DPForeceIgnoreStatus())
#endif
              ) {

            mpClientInterface->restoreOutput(a2dpOutput);
            mA2dpSuspended = false;
#ifdef MTK_AUDIO
            ALOGD("mA2dpSuspended = false, ForceUse %d, %d, phonestate %d, A2DPForeceIgnore %d",
                mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_COMMUNICATION),
                mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_RECORD), mEngine->getPhoneState(),
                mAudioPolicyVendorControl.getA2DPForeceIgnoreStatus() );
#endif
        }
    } else {
        if ((isScoConnected &&
             ((mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_COMMUNICATION) == AUDIO_POLICY_FORCE_BT_SCO) ||
              (mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_RECORD) == AUDIO_POLICY_FORCE_BT_SCO))) ||
             ((mEngine->getPhoneState() == AUDIO_MODE_IN_CALL) ||
#ifdef MTK_AUDIO
               (mEngine->getPhoneState() == AUDIO_MODE_IN_CALL_2) ||
               (mEngine->getPhoneState() == AUDIO_MODE_IN_CALL_EXTERNAL) ||
#endif
              (mEngine->getPhoneState() == AUDIO_MODE_RINGTONE))
#ifdef MTK_AUDIO
              || (mAudioPolicyVendorControl.getA2DPForeceIgnoreStatus())
#endif
              ) {

            mpClientInterface->suspendOutput(a2dpOutput);
            mA2dpSuspended = true;
#ifdef MTK_AUDIO
            ALOGD("mA2dpSuspended = true, ForceUse %d, %d, phonestate %d, A2DPForeceIgnore %d",
                mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_COMMUNICATION),
                mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_RECORD), mEngine->getPhoneState(),
                mAudioPolicyVendorControl.getA2DPForeceIgnoreStatus() );
#endif
        }
    }
}

audio_devices_t AudioPolicyManager::getNewOutputDevice(const sp<AudioOutputDescriptor>& outputDesc,
                                                       bool fromCache, bool bShareHwModule)
{
    audio_devices_t device = AUDIO_DEVICE_NONE;

    ssize_t index = mAudioPatches.indexOfKey(outputDesc->getPatchHandle());
    if (index >= 0) {
        sp<AudioPatch> patchDesc = mAudioPatches.valueAt(index);
        if (patchDesc->mUid != mUidCached) {
            ALOGV("getNewOutputDevice() device %08x forced by patch %d",
                  outputDesc->device(), outputDesc->getPatchHandle());
            return outputDesc->device();
        }
    }

#ifdef MTK_AUDIO    //  We suppose that output should refer shareHwModule to do routing
#ifdef USE_ANDROID_DEFAULT_ROUTING_STRATEGY_FOR_SHARE_HW_MODULE
        bShareHwModule = false;
#endif
#endif
    // check the following by order of priority to request a routing change if necessary:
    // 1: the strategy enforced audible is active and enforced on the output:
    //      use device for strategy enforced audible
    // 2: we are in call or the strategy phone is active on the output:
    //      use device for strategy phone
    // 3: the strategy for enforced audible is active but not enforced on the output:
    //      use the device for strategy enforced audible
    // 4: the strategy sonification is active on the output:
    //      use device for strategy sonification
    // 5: the strategy accessibility is active on the output:
    //      use device for strategy accessibility
    // 6: the strategy "respectful" sonification is active on the output:
    //      use device for strategy "respectful" sonification
    // 7: the strategy media is active on the output:
    //      use device for strategy media
    // 8: the strategy DTMF is active on the output:
    //      use device for strategy DTMF
    // 9: the strategy for beacon, a.k.a. "transmitted through speaker" is active on the output:
    //      use device for strategy t-t-s
    if (isStrategyActive(outputDesc, STRATEGY_ENFORCED_AUDIBLE, 0, 0, bShareHwModule) &&
        mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_SYSTEM) == AUDIO_POLICY_FORCE_SYSTEM_ENFORCED
#ifdef MTK_AUDIO    //  PhoneCall/RingTone is first priority
    &&(!isInCall() && mEngine->getPhoneState() != AUDIO_MODE_RINGTONE)
#endif
    ) {
        device = getDeviceForStrategy(STRATEGY_ENFORCED_AUDIBLE, fromCache);
    } else if (isInCall() ||
                    isStrategyActive(outputDesc, STRATEGY_PHONE, 0, 0, bShareHwModule)) {
        device = getDeviceForStrategy(STRATEGY_PHONE, fromCache);
    } else if (isStrategyActive(outputDesc, STRATEGY_ENFORCED_AUDIBLE, 0, 0, bShareHwModule)) {
        device = getDeviceForStrategy(STRATEGY_ENFORCED_AUDIBLE, fromCache);
    } else if (isStrategyActive(outputDesc, STRATEGY_SONIFICATION, 0, 0, bShareHwModule)
#ifdef MTK_AUDIO
           ||(isStrategyActive(outputDesc, NUM_STRATEGIES) && mEngine->getPhoneState() ==AUDIO_MODE_RINGTONE) //ALPS01092399|1851298|1923097|ALPS02325461
#endif
    ) {
        device = getDeviceForStrategy(STRATEGY_SONIFICATION, fromCache);
    } else if (isStrategyActive(outputDesc, STRATEGY_ACCESSIBILITY, 0, 0, bShareHwModule)) {
        device = getDeviceForStrategy(STRATEGY_ACCESSIBILITY, fromCache);
    } else if (isStrategyActive(outputDesc, STRATEGY_SONIFICATION_RESPECTFUL, 0, 0, bShareHwModule)) {
        device = getDeviceForStrategy(STRATEGY_SONIFICATION_RESPECTFUL, fromCache);
    } else if (isStrategyActive(outputDesc, STRATEGY_MEDIA, 0, 0, bShareHwModule)) {
        device = getDeviceForStrategy(STRATEGY_MEDIA, fromCache);
    } else if (isStrategyActive(outputDesc, STRATEGY_DTMF, 0, 0, bShareHwModule)) {
        device = getDeviceForStrategy(STRATEGY_DTMF, fromCache);
    } else if (isStrategyActive(outputDesc, STRATEGY_TRANSMITTED_THROUGH_SPEAKER, 0, 0, bShareHwModule)) {
        device = getDeviceForStrategy(STRATEGY_TRANSMITTED_THROUGH_SPEAKER, fromCache);
    } else if (isStrategyActive(outputDesc, STRATEGY_REROUTING, 0, 0, bShareHwModule)) {
        device = getDeviceForStrategy(STRATEGY_REROUTING, fromCache);
    }

    ALOGV("getNewOutputDevice() selected device %x", device);
    return device;
}

audio_devices_t AudioPolicyManager::getNewInputDevice(audio_io_handle_t input)
{
    sp<AudioInputDescriptor> inputDesc = mInputs.valueFor(input);

    ssize_t index = mAudioPatches.indexOfKey(inputDesc->getPatchHandle());
    if (index >= 0) {
        sp<AudioPatch> patchDesc = mAudioPatches.valueAt(index);
        if (patchDesc->mUid != mUidCached) {
            ALOGV("getNewInputDevice() device %08x forced by patch %d",
                  inputDesc->mDevice, inputDesc->getPatchHandle());
            return inputDesc->mDevice;
        }
    }

    audio_devices_t device = getDeviceAndMixForInputSource(inputDesc->inputSource());

    return device;
}

bool AudioPolicyManager::streamsMatchForvolume(audio_stream_type_t stream1,
                                               audio_stream_type_t stream2) {
    return ((stream1 == stream2) ||
            ((stream1 == AUDIO_STREAM_ACCESSIBILITY) && (stream2 == AUDIO_STREAM_MUSIC)) ||
            ((stream1 == AUDIO_STREAM_MUSIC) && (stream2 == AUDIO_STREAM_ACCESSIBILITY)));
}

uint32_t AudioPolicyManager::getStrategyForStream(audio_stream_type_t stream) {
    return (uint32_t)getStrategy(stream);
}

audio_devices_t AudioPolicyManager::getDevicesForStream(audio_stream_type_t stream) {
    // By checking the range of stream before calling getStrategy, we avoid
    // getStrategy's behavior for invalid streams.  getStrategy would do a ALOGE
    // and then return STRATEGY_MEDIA, but we want to return the empty set.
    if (stream < (audio_stream_type_t) 0 || stream >= AUDIO_STREAM_PUBLIC_CNT) {
        return AUDIO_DEVICE_NONE;
    }
    audio_devices_t devices = AUDIO_DEVICE_NONE;
    for (int curStream = 0; curStream < AUDIO_STREAM_FOR_POLICY_CNT; curStream++) {
        if (!streamsMatchForvolume(stream, (audio_stream_type_t)curStream)) {
            continue;
        }
        routing_strategy curStrategy = getStrategy((audio_stream_type_t)curStream);
        audio_devices_t curDevices =
                getDeviceForStrategy((routing_strategy)curStrategy, true /*fromCache*/);
        SortedVector<audio_io_handle_t> outputs = getOutputsForDevice(curDevices, mOutputs);
        for (size_t i = 0; i < outputs.size(); i++) {
            sp<AudioOutputDescriptor> outputDesc = mOutputs.valueFor(outputs[i]);
            if (outputDesc->isStreamActive((audio_stream_type_t)curStream)) {
#ifdef MTK_CROSSMOUNT_SUPPORT //ALPS02193613
                if (mAudioPolicyVendorControl.getCrossMountLocalPlayback() && (outputDesc->device() == (AUDIO_DEVICE_OUT_REMOTE_SUBMIX|AUDIO_DEVICE_OUT_SPEAKER))) {
                    ALOGD("%s change [0x%x] to [0x%x]", __FUNCTION__, devices, AUDIO_DEVICE_OUT_REMOTE_SUBMIX);
                    curDevices |= AUDIO_DEVICE_OUT_REMOTE_SUBMIX;
                } else {
                    curDevices |= outputDesc->device();
                }
#else
                curDevices |= outputDesc->device();
#endif
            }
        }
        devices |= curDevices;
    }

    /*Filter SPEAKER_SAFE out of results, as AudioService doesn't know about it
      and doesn't really need to.*/
    if (devices & AUDIO_DEVICE_OUT_SPEAKER_SAFE) {
        devices |= AUDIO_DEVICE_OUT_SPEAKER;
        devices &= ~AUDIO_DEVICE_OUT_SPEAKER_SAFE;
    }
    return devices;
}

routing_strategy AudioPolicyManager::getStrategy(audio_stream_type_t stream) const
{
    ALOG_ASSERT(stream != AUDIO_STREAM_PATCH,"getStrategy() called for AUDIO_STREAM_PATCH");
    return mEngine->getStrategyForStream(stream);
}

uint32_t AudioPolicyManager::getStrategyForAttr(const audio_attributes_t *attr) {
    // flags to strategy mapping
    if ((attr->flags & AUDIO_FLAG_BEACON) == AUDIO_FLAG_BEACON) {
        return (uint32_t) STRATEGY_TRANSMITTED_THROUGH_SPEAKER;
    }
    if ((attr->flags & AUDIO_FLAG_AUDIBILITY_ENFORCED) == AUDIO_FLAG_AUDIBILITY_ENFORCED) {
        return (uint32_t) STRATEGY_ENFORCED_AUDIBLE;
    }
    // usage to strategy mapping
    return static_cast<uint32_t>(mEngine->getStrategyForUsage(attr->usage));
}

void AudioPolicyManager::handleNotificationRoutingForStream(audio_stream_type_t stream) {
    switch(stream) {
    case AUDIO_STREAM_MUSIC:
        checkOutputForStrategy(STRATEGY_SONIFICATION_RESPECTFUL);
        updateDevicesAndOutputs();
        break;
    default:
        break;
    }
}

uint32_t AudioPolicyManager::handleEventForBeacon(int event) {

    // skip beacon mute management if a dedicated TTS output is available
    if (mTtsOutputAvailable) {
        return 0;
    }

    switch(event) {
    case STARTING_OUTPUT:
        mBeaconMuteRefCount++;
        break;
    case STOPPING_OUTPUT:
        if (mBeaconMuteRefCount > 0) {
            mBeaconMuteRefCount--;
        }
        break;
    case STARTING_BEACON:
        mBeaconPlayingRefCount++;
        break;
    case STOPPING_BEACON:
        if (mBeaconPlayingRefCount > 0) {
            mBeaconPlayingRefCount--;
        }
        break;
    }

    if (mBeaconMuteRefCount > 0) {
        // any playback causes beacon to be muted
        return setBeaconMute(true);
    } else {
        // no other playback: unmute when beacon starts playing, mute when it stops
        return setBeaconMute(mBeaconPlayingRefCount == 0);
    }
}

uint32_t AudioPolicyManager::setBeaconMute(bool mute) {
#ifdef MTK_AUDIO
    MTK_ALOGV("setBeaconMute(%d) mBeaconMuteRefCount=%d mBeaconPlayingRefCount=%d",
            mute, mBeaconMuteRefCount, mBeaconPlayingRefCount);
#else
    ALOGV("setBeaconMute(%d) mBeaconMuteRefCount=%d mBeaconPlayingRefCount=%d",
            mute, mBeaconMuteRefCount, mBeaconPlayingRefCount);
#endif
    // keep track of muted state to avoid repeating mute/unmute operations
    if (mBeaconMuted != mute) {
        // mute/unmute AUDIO_STREAM_TTS on all outputs
        ALOGV("\t muting %d", mute);
        uint32_t maxLatency = 0;
        for (size_t i = 0; i < mOutputs.size(); i++) {
            sp<SwAudioOutputDescriptor> desc = mOutputs.valueAt(i);
            setStreamMute(AUDIO_STREAM_TTS, mute/*on*/,
                    desc,
                    0 /*delay*/, AUDIO_DEVICE_NONE);
            const uint32_t latency = desc->latency() * 2;
            if (latency > maxLatency) {
                maxLatency = latency;
            }
        }
        mBeaconMuted = mute;
        return maxLatency;
    }
    return 0;
}

//<MTK_AUDIO_ADD
//audio_devices_t AudioPolicyManager::getDeviceForStrategy(routing_strategy strategy,
                                                             //bool fromCache)
audio_devices_t AudioPolicyManager::getDeviceForStrategy(routing_strategy strategy,
                                                         bool fromCache, audio_output_flags_t flags)
//MTK_AUDIO_ADD>
{
    // Routing
    // see if we have an explicit route
    // scan the whole RouteMap, for each entry, convert the stream type to a strategy
    // (getStrategy(stream)).
    // if the strategy from the stream type in the RouteMap is the same as the argument above,
    // and activity count is non-zero
    // the device = the device from the descriptor in the RouteMap, and exit.
    for (size_t routeIndex = 0; routeIndex < mOutputRoutes.size(); routeIndex++) {
        sp<SessionRoute> route = mOutputRoutes.valueAt(routeIndex);
        routing_strategy routeStrategy = getStrategy(route->mStreamType);
        if ((routeStrategy == strategy) && route->isActive()) {
            return route->mDeviceDescriptor->type();
        }
    }

    if (fromCache) {
#ifdef MTK_AUDIO
        MTK_ALOGVV("getDeviceForStrategy() from cache strategy %d, device %x",
              strategy, mDeviceForStrategy[strategy]);
#else
        ALOGVV("getDeviceForStrategy() from cache strategy %d, device %x",
              strategy, mDeviceForStrategy[strategy]);
#endif
        return mDeviceForStrategy[strategy];
    }
    return mEngine->getDeviceForStrategy(strategy, flags);
}

void AudioPolicyManager::updateDevicesAndOutputs()
{
    for (int i = 0; i < NUM_STRATEGIES; i++) {
        mDeviceForStrategy[i] = getDeviceForStrategy((routing_strategy)i, false /*fromCache*/);
    }
    mPreviousOutputs = mOutputs;
}

uint32_t AudioPolicyManager::checkDeviceMuteStrategies(sp<AudioOutputDescriptor> outputDesc,
                                                       audio_devices_t prevDevice,
                                                       uint32_t delayMs)
{
    // mute/unmute strategies using an incompatible device combination
    // if muting, wait for the audio in pcm buffer to be drained before proceeding
    // if unmuting, unmute only after the specified delay
    if (outputDesc->isDuplicated()) {
        return 0;
    }

    uint32_t muteWaitMs = 0;
    audio_devices_t device = outputDesc->device();
#ifdef MTK_AUDIO    //  There is no active stream when incall, but we wish to mute before routing at incall mode
    bool shouldMute = (outputDesc->isActive()||isInCall()) && (popcount(device) >= 2);
#else
    bool shouldMute = outputDesc->isActive() && (popcount(device) >= 2);
#endif

    for (size_t i = 0; i < NUM_STRATEGIES; i++) {
        audio_devices_t curDevice = getDeviceForStrategy((routing_strategy)i, false /*fromCache*/);
        curDevice = curDevice & outputDesc->supportedDevices();
        bool mute = shouldMute && (curDevice & device) && (curDevice != device);
        bool doMute = false;

        if (mute && !outputDesc->mStrategyMutedByDevice[i]) {
            doMute = true;
            outputDesc->mStrategyMutedByDevice[i] = true;
        } else if (!mute && outputDesc->mStrategyMutedByDevice[i]){
            doMute = true;
            outputDesc->mStrategyMutedByDevice[i] = false;
        }
        if (doMute) {
            for (size_t j = 0; j < mOutputs.size(); j++) {
                sp<AudioOutputDescriptor> desc = mOutputs.valueAt(j);
                // skip output if it does not share any device with current output
                if ((desc->supportedDevices() & outputDesc->supportedDevices())
                        == AUDIO_DEVICE_NONE) {
                    continue;
                }
                ALOGVV("checkDeviceMuteStrategies() %s strategy %zu (curDevice %04x)",
                      mute ? "muting" : "unmuting", i, curDevice);
                setStrategyMute((routing_strategy)i, mute, desc, mute ? 0 : delayMs);
                if (isStrategyActive(desc, (routing_strategy)i)) {
                    if (mute) {
                        // FIXME: should not need to double latency if volume could be applied
                        // immediately by the audioflinger mixer. We must account for the delay
                        // between now and the next time the audioflinger thread for this output
                        // will process a buffer (which corresponds to one buffer size,
                        // usually 1/2 or 1/4 of the latency).
                        if (muteWaitMs < desc->latency() * 2) {
                            muteWaitMs = desc->latency() * 2;
                        }
                    }
                }
            }
        }
    }

    // temporary mute output if device selection changes to avoid volume bursts due to
    // different per device volumes
#ifdef MTK_AUDIO    //  There is no active stream when incall, but we wish to mute before routing at incall mode
    if ((outputDesc->isActive()||isInCall()) && (device != prevDevice)
    && (prevDevice != AUDIO_DEVICE_NONE)
    && !(prevDevice == AUDIO_DEVICE_OUT_EARPIECE && !isInCall() && !outputDesc->isStreamActive(AUDIO_STREAM_MUSIC)))
#else
    if (outputDesc->isActive() && (device != prevDevice))
#endif
    {
        if (muteWaitMs < outputDesc->latency() * 2) {
            muteWaitMs = outputDesc->latency() * 2;
        }
        for (size_t i = 0; i < NUM_STRATEGIES; i++) {
#ifdef MTK_AUDIO    //  There is no active stream when incall, but we wish to mute before routing at incall mode
            if ((isStrategyActive(outputDesc, (routing_strategy)i)) || (i == STRATEGY_PHONE && isInCall()))
#else
            if (isStrategyActive(outputDesc, (routing_strategy)i))
#endif
            {
                setStrategyMute((routing_strategy)i, true, outputDesc);
                // do tempMute unmute after twice the mute wait time
                setStrategyMute((routing_strategy)i, false, outputDesc,
                                muteWaitMs *2, device);
            }
        }
    }

    // wait for the PCM output buffers to empty before proceeding with the rest of the command
    if (muteWaitMs > delayMs) {
        muteWaitMs -= delayMs;
        usleep(muteWaitMs * 1000);
        return muteWaitMs;
    }
    return 0;
}

uint32_t AudioPolicyManager::setOutputDevice(const sp<AudioOutputDescriptor>& outputDesc,
                                             audio_devices_t device,
                                             bool force,
                                             int delayMs,
                                             audio_patch_handle_t *patchHandle,
                                             const char* address)
{
#ifdef MTK_AUDIO    //  Show enough log to debug
    for (size_t i = 0; i < mOutputs.size(); i++) {
        sp<AudioOutputDescriptor> outputdesc = mOutputs.valueAt(i);
        if (outputDesc == outputdesc) {
            ALOGV("setOutputDevice() mIoHandle %d device %04x(%04x)(%04x) delayMs %d force %d size %zu", mOutputs.keyAt(i), device, outputDesc->mDevice, outputDesc->supportedDevices(), delayMs, force, mOutputs.size());
            break;
        }
        if (i == mOutputs.size())
            ALOGV("setOutputDevice() device %04x delayMs %d force %d outputsize %zu", device, delayMs, force, mOutputs.size());
    }
#else
    ALOGV("setOutputDevice() device %04x delayMs %d", device, delayMs);
#endif
    AudioParameter param;
    uint32_t muteWaitMs;

    if (outputDesc->isDuplicated()) {
#ifdef MTK_AUDIO
        AudioParameter param;
        param.addInt(String8("AudioFlinger_routing"), (int)device);
        for (size_t i = 0; i < mOutputs.size(); i++) {
            sp<SwAudioOutputDescriptor> outputdesc = mOutputs.valueAt(i);
            if (outputDesc == outputdesc) {
                mpClientInterface->setParameters(outputdesc->mIoHandle, param.toString(), delayMs);
                break;
            }
        }
#endif
        muteWaitMs = setOutputDevice(outputDesc->subOutput1(), device, force, delayMs);
        muteWaitMs += setOutputDevice(outputDesc->subOutput2(), device, force, delayMs);
        return muteWaitMs;
    }
#ifdef MTK_AUDIO    //  Try to log caller wanted device
    audio_devices_t wanted_device = device;
#endif
    // no need to proceed if new device is not AUDIO_DEVICE_NONE and not supported by current
    // output profile
    if ((device != AUDIO_DEVICE_NONE) &&
#if defined(MTK_AUDIO) && defined(SUPPORT_ANDROID_FM_PLAYER)
        // When Device Connected (ex: A2DP) in FM Playback, we need to callback to frameworks
        (!force) &&
#endif
            ((device & outputDesc->supportedDevices()) == 0)) {
        return 0;
    }

    // filter devices according to output selected
    device = (audio_devices_t)(device & outputDesc->supportedDevices());

    audio_devices_t prevDevice = outputDesc->mDevice;
#ifndef MTK_AUDIO
    ALOGV("setOutputDevice() prevDevice 0x%04x", prevDevice);
#endif

    if (device != AUDIO_DEVICE_NONE) {
        outputDesc->mDevice = device;
    }
#ifdef MTK_AUDIO
    audio_devices_t MuteprevDevice;
    if (outputDesc == mPrimaryOutput) {
        MuteprevDevice = outputDesc->mMutePrevDevice;
        if (outputDesc->mMutePrevDevice == AUDIO_DEVICE_NONE)// Initial handler
            MuteprevDevice = prevDevice;
        MTK_ALOGV("setOutputDevice() mMutePrevDevice %04x", MuteprevDevice);
        outputDesc->mMutePrevDevice = device;
    } else {
        MuteprevDevice = prevDevice;
    }

    if ((device & AUDIO_DEVICE_OUT_WIRED_HEADSET) && (prevDevice & AUDIO_DEVICE_OUT_WIRED_HEADPHONE)) {
       // ALPS00568167: headphone -> headset. It will mute music twice.
       muteWaitMs = 0;
    } else if (outputDesc->mOutputFirstActive == true && (outputDesc != mPrimaryOutput || !isInCall())){ //if stream is first active, skip MuteStrategies, and reset flag
       muteWaitMs = 0;
       outputDesc->mOutputFirstActive = false;
    } else {
       muteWaitMs = checkDeviceMuteStrategies(outputDesc, MuteprevDevice, delayMs);
    }
#else
    muteWaitMs = checkDeviceMuteStrategies(outputDesc, prevDevice, delayMs);
#endif

    // Do not change the routing if:
    //      the requested device is AUDIO_DEVICE_NONE
    //      OR the requested device is the same as current device
    //  AND force is not specified
    //  AND the output is connected by a valid audio patch.
    // Doing this check here allows the caller to call setOutputDevice() without conditions
    if ((device == AUDIO_DEVICE_NONE || device == prevDevice) &&
        !force &&
        outputDesc->getPatchHandle() != 0) {
        ALOGV("setOutputDevice() setting same device 0x%04x or null device", device);
        return muteWaitMs;
    }
#ifdef MTK_AUDIO
    // Do not change the routing if:
    //  the requested device is AUDIO_DEVICE_NONE(Need reset output)
    //  AND force is not specified
    //  AND there is no audio patch(Already reseted)
    if (device == AUDIO_DEVICE_NONE &&
        !force &&
        outputDesc->getPatchHandle() == 0) {
        ALOGV("setOutputDevice() already resetOutput, don't need routing");
        return muteWaitMs;
    }
#endif
    ALOGV("setOutputDevice() changing device");

    // do the routing
    if (device == AUDIO_DEVICE_NONE) {
        resetOutputDevice(outputDesc, delayMs, NULL);
    } else {
        DeviceVector deviceList;
        if ((address == NULL) || (strlen(address) == 0)) {
            deviceList = mAvailableOutputDevices.getDevicesFromType(device);
        } else {
            deviceList = mAvailableOutputDevices.getDevicesFromTypeAddr(device, String8(address));
        }

        if (!deviceList.isEmpty()) {
            struct audio_patch patch;
            outputDesc->toAudioPortConfig(&patch.sources[0]);
            patch.num_sources = 1;
            patch.num_sinks = 0;
            for (size_t i = 0; i < deviceList.size() && i < AUDIO_PATCH_PORTS_MAX; i++) {
                deviceList.itemAt(i)->toAudioPortConfig(&patch.sinks[i]);
                patch.num_sinks++;
            }
            ssize_t index;
            if (patchHandle && *patchHandle != AUDIO_PATCH_HANDLE_NONE) {
                index = mAudioPatches.indexOfKey(*patchHandle);
            } else {
                index = mAudioPatches.indexOfKey(outputDesc->getPatchHandle());
            }
            sp< AudioPatch> patchDesc;
            audio_patch_handle_t afPatchHandle = AUDIO_PATCH_HANDLE_NONE;
            if (index >= 0) {
                patchDesc = mAudioPatches.valueAt(index);
                afPatchHandle = patchDesc->mAfPatchHandle;
            }

            status_t status = mpClientInterface->createAudioPatch(&patch,
                                                                   &afPatchHandle,
                                                                   delayMs);
#ifndef MTK_AUDIO
            ALOGV("setOutputDevice() createAudioPatch returned %d patchHandle %d"
                    "num_sources %d num_sinks %d",
                                       status, afPatchHandle, patch.num_sources, patch.num_sinks);
#endif
            if (status == NO_ERROR) {
                if (index < 0) {
                    patchDesc = new AudioPatch(&patch, mUidCached);
                    addAudioPatch(patchDesc->mHandle, patchDesc);
                } else {
                    patchDesc->mPatch = patch;
                }
                patchDesc->mAfPatchHandle = afPatchHandle;
                if (patchHandle) {
                    *patchHandle = patchDesc->mHandle;
                }
                outputDesc->setPatchHandle(patchDesc->mHandle);
                nextAudioPortGeneration();
                mpClientInterface->onAudioPatchListUpdate();
            }
#ifdef MTK_AUDIO
            else {
                ALOGV("setOutputDevice() createAudioPatch returned %d patchHandle %d"
                    "num_sources %d num_sinks %d",
                                       status, afPatchHandle, patch.num_sources, patch.num_sinks);
            }
#endif
        }

        // inform all input as well
        for (size_t i = 0; i < mInputs.size(); i++) {
            const sp<AudioInputDescriptor>  inputDescriptor = mInputs.valueAt(i);
            if (!is_virtual_input_device(inputDescriptor->mDevice)) {
                AudioParameter inputCmd = AudioParameter();
                ALOGV("%s: inform input %d of device:%d", __func__,
                      inputDescriptor->mIoHandle, device);
                inputCmd.addInt(String8(AudioParameter::keyRouting),device);
                mpClientInterface->setParameters(inputDescriptor->mIoHandle,
                                                 inputCmd.toString(),
                                                 delayMs);
            }
        }
    }

#ifdef MTK_AUDIO
    // ALPS02453417
    if ((wanted_device != AUDIO_DEVICE_NONE) && (device == AUDIO_DEVICE_NONE)) {
        // Ex: SUPPORT_ANDROID_FM_PLAYER && force = true
        // It should return at beginning
        return 0;
    } else if ((wanted_device == AUDIO_DEVICE_NONE) && (outputDesc->supportedDevices() & AUDIO_DEVICE_OUT_SPEAKER) == 0) {
        // Routing to none (will use AUDIO_DEVICE_OUT_SPEAKER volume),
        // but the output don't support AUDIO_DEVICE_OUT_SPEAKER
        return 0;
    }
#endif
    // update stream volumes according to new device
    applyStreamVolumes(outputDesc, device, delayMs);

    return muteWaitMs;
}

status_t AudioPolicyManager::resetOutputDevice(const sp<AudioOutputDescriptor>& outputDesc,
                                               int delayMs,
                                               audio_patch_handle_t *patchHandle)
{
    ssize_t index;
    if (patchHandle) {
        index = mAudioPatches.indexOfKey(*patchHandle);
    } else {
        index = mAudioPatches.indexOfKey(outputDesc->getPatchHandle());
    }
    if (index < 0) {
        return INVALID_OPERATION;
    }
    sp< AudioPatch> patchDesc = mAudioPatches.valueAt(index);
    status_t status = mpClientInterface->releaseAudioPatch(patchDesc->mAfPatchHandle, delayMs);
    ALOGV("resetOutputDevice() releaseAudioPatch returned %d", status);
    outputDesc->setPatchHandle(AUDIO_PATCH_HANDLE_NONE);
    removeAudioPatch(patchDesc->mHandle);
    nextAudioPortGeneration();
    mpClientInterface->onAudioPatchListUpdate();
    return status;
}

status_t AudioPolicyManager::setInputDevice(audio_io_handle_t input,
                                            audio_devices_t device,
                                            bool force,
                                            audio_patch_handle_t *patchHandle)
{
    status_t status = NO_ERROR;

    sp<AudioInputDescriptor> inputDesc = mInputs.valueFor(input);
    if ((device != AUDIO_DEVICE_NONE) && ((device != inputDesc->mDevice) || force)) {
        inputDesc->mDevice = device;

        DeviceVector deviceList = mAvailableInputDevices.getDevicesFromType(device);
        if (!deviceList.isEmpty()) {
            struct audio_patch patch;
            inputDesc->toAudioPortConfig(&patch.sinks[0]);
            // AUDIO_SOURCE_HOTWORD is for internal use only:
            // handled as AUDIO_SOURCE_VOICE_RECOGNITION by the audio HAL
            if (patch.sinks[0].ext.mix.usecase.source == AUDIO_SOURCE_HOTWORD &&
                    !inputDesc->isSoundTrigger()) {
                patch.sinks[0].ext.mix.usecase.source = AUDIO_SOURCE_VOICE_RECOGNITION;
            }
            patch.num_sinks = 1;
            //only one input device for now
            deviceList.itemAt(0)->toAudioPortConfig(&patch.sources[0]);
            patch.num_sources = 1;
            ssize_t index;
            if (patchHandle && *patchHandle != AUDIO_PATCH_HANDLE_NONE) {
                index = mAudioPatches.indexOfKey(*patchHandle);
            } else {
                index = mAudioPatches.indexOfKey(inputDesc->getPatchHandle());
            }
            sp< AudioPatch> patchDesc;
            audio_patch_handle_t afPatchHandle = AUDIO_PATCH_HANDLE_NONE;
            if (index >= 0) {
                patchDesc = mAudioPatches.valueAt(index);
                afPatchHandle = patchDesc->mAfPatchHandle;
            }

            status_t status = mpClientInterface->createAudioPatch(&patch,
                                                                  &afPatchHandle,
                                                                  0);
            ALOGV("setInputDevice() createAudioPatch returned %d patchHandle %d",
                                                                          status, afPatchHandle);
            if (status == NO_ERROR) {
                if (index < 0) {
                    patchDesc = new AudioPatch(&patch, mUidCached);
                    addAudioPatch(patchDesc->mHandle, patchDesc);
                } else {
                    patchDesc->mPatch = patch;
                }
                patchDesc->mAfPatchHandle = afPatchHandle;
                if (patchHandle) {
                    *patchHandle = patchDesc->mHandle;
                }
                inputDesc->setPatchHandle(patchDesc->mHandle);
                nextAudioPortGeneration();
                mpClientInterface->onAudioPatchListUpdate();
            }
        }
    }
    return status;
}

status_t AudioPolicyManager::resetInputDevice(audio_io_handle_t input,
                                              audio_patch_handle_t *patchHandle)
{
    sp<AudioInputDescriptor> inputDesc = mInputs.valueFor(input);
    ssize_t index;
    if (patchHandle) {
        index = mAudioPatches.indexOfKey(*patchHandle);
    } else {
        index = mAudioPatches.indexOfKey(inputDesc->getPatchHandle());
    }
    if (index < 0) {
        return INVALID_OPERATION;
    }
    sp< AudioPatch> patchDesc = mAudioPatches.valueAt(index);
    status_t status = mpClientInterface->releaseAudioPatch(patchDesc->mAfPatchHandle, 0);
    ALOGV("resetInputDevice() releaseAudioPatch returned %d", status);
    inputDesc->setPatchHandle(AUDIO_PATCH_HANDLE_NONE);
    removeAudioPatch(patchDesc->mHandle);
    nextAudioPortGeneration();
    mpClientInterface->onAudioPatchListUpdate();
    return status;
}

sp<IOProfile> AudioPolicyManager::getInputProfile(audio_devices_t device,
                                                  String8 address,
                                                  uint32_t& samplingRate,
                                                  audio_format_t& format,
                                                  audio_channel_mask_t& channelMask,
                                                  audio_input_flags_t flags)
{
    // Choose an input profile based on the requested capture parameters: select the first available
    // profile supporting all requested parameters.
    //
    // TODO: perhaps isCompatibleProfile should return a "matching" score so we can return
    // the best matching profile, not the first one.

    for (size_t i = 0; i < mHwModules.size(); i++)
    {
        if (mHwModules[i]->mHandle == 0) {
            continue;
        }
        for (size_t j = 0; j < mHwModules[i]->mInputProfiles.size(); j++)
        {
            sp<IOProfile> profile = mHwModules[i]->mInputProfiles[j];
            // profile->log();
            if (profile->isCompatibleProfile(device, address, samplingRate,
                                             &samplingRate /*updatedSamplingRate*/,
                                             format,
                                             &format /*updatedFormat*/,
                                             channelMask,
                                             &channelMask /*updatedChannelMask*/,
                                             (audio_output_flags_t) flags)) {

                return profile;
            }
        }
    }
    return NULL;
}


audio_devices_t AudioPolicyManager::getDeviceAndMixForInputSource(audio_source_t inputSource,
                                                                  AudioMix **policyMix)
{
    audio_devices_t availableDeviceTypes = mAvailableInputDevices.types() & ~AUDIO_DEVICE_BIT_IN;
    audio_devices_t selectedDeviceFromMix =
           mPolicyMixes.getDeviceAndMixForInputSource(inputSource, availableDeviceTypes, policyMix);

    if (selectedDeviceFromMix != AUDIO_DEVICE_NONE) {
        return selectedDeviceFromMix;
    }
    return getDeviceForInputSource(inputSource);
}

audio_devices_t AudioPolicyManager::getDeviceForInputSource(audio_source_t inputSource)
{
    for (size_t routeIndex = 0; routeIndex < mInputRoutes.size(); routeIndex++) {
         sp<SessionRoute> route = mInputRoutes.valueAt(routeIndex);
         if (inputSource == route->mSource && route->isActive()) {
             return route->mDeviceDescriptor->type();
         }
     }

     return mEngine->getDeviceForInputSource(inputSource);
}

float AudioPolicyManager::computeVolume(audio_stream_type_t stream,
                                        int index,
                                        audio_devices_t device)
{
#ifdef MTK_AUDIO    //  Use custome volume mapping
        float volumeDB;
        if (mAudioPolicyVendorControl.getCustomVolumeStatus()) {
            volumeDB = Volume::AmplToDb(computeCustomVolume(stream, index, device));
        } else {
            ALOGW("%s,not Customer Volume, Using Android Volume Curve",__FUNCTION__);
            volumeDB = mVolumeCurves->volIndexToDb(stream, Volume::getDeviceCategory(device), index);
        }
        MTK_ALOGV("%s streamtype [%d],index [%d],device [0x%x], volumeDb [%f]",__FUNCTION__,stream,index,device,volumeDB);
#else
    float volumeDB = mVolumeCurves->volIndexToDb(stream, Volume::getDeviceCategory(device), index);
#endif
    // if a headset is connected, apply the following rules to ring tones and notifications
    // to avoid sound level bursts in user's ears:
    // - always attenuate notifications volume by 6dB
    // - attenuate ring tones volume by 6dB unless music is not playing and
    // speaker is part of the select devices
    // - if music is playing, always limit the volume to current music volume,
    // with a minimum threshold at -36dB so that notification is always perceived.
    const routing_strategy stream_strategy = getStrategy(stream);
#ifdef MTK_AUDIO
    audio_devices_t Streamdevices = getDeviceForStrategy(stream_strategy, true /*fromCache*/);
    if ((device & Streamdevices) && (isStreamActive(AUDIO_STREAM_MUSIC, SONIFICATION_HEADSET_MUSIC_DELAY) ||
            mLimitRingtoneVolume)) {
        device = Streamdevices;
    }
#endif
    if ((device & (AUDIO_DEVICE_OUT_BLUETOOTH_A2DP |
            AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_HEADPHONES |
            AUDIO_DEVICE_OUT_WIRED_HEADSET |
            AUDIO_DEVICE_OUT_WIRED_HEADPHONE)) &&
        ((stream_strategy == STRATEGY_SONIFICATION)
                || (stream_strategy == STRATEGY_SONIFICATION_RESPECTFUL)
                || (stream == AUDIO_STREAM_SYSTEM)
                || ((stream_strategy == STRATEGY_ENFORCED_AUDIBLE) &&
                    (mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_SYSTEM) == AUDIO_POLICY_FORCE_NONE))) &&
            mVolumeCurves->canBeMuted(stream)) {
        // when the phone is ringing we must consider that music could have been paused just before
        // by the music application and behave as if music was active if the last music track was
        // just stopped
#ifdef MTK_AUDIO
        uint32_t mask = (uint32_t)(device) & (uint32_t)(~AUDIO_DEVICE_OUT_SPEAKER);
        device = (audio_devices_t)(mask) ; // use correct volume index
#endif
        if (isStreamActive(AUDIO_STREAM_MUSIC, SONIFICATION_HEADSET_MUSIC_DELAY) ||
                mLimitRingtoneVolume) {
            volumeDB += SONIFICATION_HEADSET_VOLUME_FACTOR_DB;
            audio_devices_t musicDevice = getDeviceForStrategy(STRATEGY_MEDIA, true /*fromCache*/);
            float musicVolDB = computeVolume(AUDIO_STREAM_MUSIC,
                                             mVolumeCurves->getVolumeIndex(AUDIO_STREAM_MUSIC,
                                                                              musicDevice),
                                             musicDevice);
            float minVolDB = (musicVolDB > SONIFICATION_HEADSET_VOLUME_MIN_DB) ?
                    musicVolDB : SONIFICATION_HEADSET_VOLUME_MIN_DB;
            if (volumeDB > minVolDB) {
                volumeDB = minVolDB;
                ALOGV("computeVolume limiting volume to %f musicVol %f", minVolDB, musicVolDB);
            }
            if (device & (AUDIO_DEVICE_OUT_BLUETOOTH_A2DP |
                    AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_HEADPHONES)) {
                // on A2DP, also ensure notification volume is not too low compared to media when
                // intended to be played
                if ((volumeDB > -96.0f) &&
                        (musicVolDB - SONIFICATION_A2DP_MAX_MEDIA_DIFF_DB > volumeDB)) {
                    ALOGV("computeVolume increasing volume for stream=%d device=0x%X from %f to %f",
                            stream, device,
                            volumeDB, musicVolDB - SONIFICATION_A2DP_MAX_MEDIA_DIFF_DB);
                    volumeDB = musicVolDB - SONIFICATION_A2DP_MAX_MEDIA_DIFF_DB;
                }
            }
        } else if ((Volume::getDeviceForVolume(device) != AUDIO_DEVICE_OUT_SPEAKER) ||
                stream_strategy != STRATEGY_SONIFICATION) {
            volumeDB += SONIFICATION_HEADSET_VOLUME_FACTOR_DB;
        }
    }
#ifdef MTK_AUDIO
    if ((device & AUDIO_DEVICE_OUT_AUX_DIGITAL) &&
        ((stream_strategy == STRATEGY_SONIFICATION)|| (stream_strategy == STRATEGY_SONIFICATION_RESPECTFUL))) {
        ALOGD("AUDIO_DEVICE_OUT_AUX_DIGITAL device = 0x%x stream_strategy = %d",device,stream_strategy);
        if (isStreamActive(AUDIO_STREAM_MUSIC, SONIFICATION_HEADSET_MUSIC_DELAY)) {
            if (volumeDB < SONIFICATION_AUX_DIGITAL_VOLUME_FACTOR_DB) {
                while(volumeDB < SONIFICATION_AUX_DIGITAL_VOLUME_FACTOR_DB)
                    volumeDB = volumeDB + (SONIFICATION_AUX_DIGITAL_VOLUME_FACTOR_DB*(-1));
            }
        }
    }
#endif
    return volumeDB;
}

status_t AudioPolicyManager::checkAndSetVolume(audio_stream_type_t stream,
                                                   int index,
                                                   const sp<AudioOutputDescriptor>& outputDesc,
                                                   audio_devices_t device,
                                                   int delayMs,
                                                   bool force)
{
#if defined(MTK_AUDIO)
    audio_devices_t originDevice = device;
#endif

    // do not change actual stream volume if the stream is muted
    if (outputDesc->mMuteCount[stream] != 0) {
        ALOGVV("checkAndSetVolume() stream %d muted count %d",
              stream, outputDesc->mMuteCount[stream]);
        return NO_ERROR;
    }
    audio_policy_forced_cfg_t forceUseForComm =
            mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_COMMUNICATION);
    // do not change in call volume if bluetooth is connected and vice versa
    if ((stream == AUDIO_STREAM_VOICE_CALL && forceUseForComm == AUDIO_POLICY_FORCE_BT_SCO) ||
        (stream == AUDIO_STREAM_BLUETOOTH_SCO && forceUseForComm != AUDIO_POLICY_FORCE_BT_SCO)) {
        ALOGV("checkAndSetVolume() cannot set stream %d volume with force use = %d for comm",
             stream, forceUseForComm);
        return INVALID_OPERATION;
    }

#ifdef MTK_AUDIO
    if ((device != AUDIO_DEVICE_NONE) && !outputDesc->isDuplicated()) {
        if ((device & outputDesc->supportedDevices()) == 0) {
            ALOGE("%s invalid set device [0x%x] volume to mId[%d](device support 0x%x), use [0x%x]",
                __FUNCTION__, device, outputDesc->getId(), outputDesc->supportedDevices(), outputDesc->device());
            device = AUDIO_DEVICE_NONE;
        }
    }
#endif
    if (device == AUDIO_DEVICE_NONE) {
        device = outputDesc->device();
    }
#if defined(MTK_AUDIO) && defined(MTK_AUDIO_GAIN_TABLE)
    float volumeDb = computeVolume(stream, index, device, outputDesc);
#else
    float volumeDb = computeVolume(stream, index, device);
#endif
    if (outputDesc->isFixedVolume(device)) {
        volumeDb = 0.0f;
    }

#ifdef MTK_AUDIO
     // for VT notify tone when incoming call. it's volume will be adusted in hardware.
     if ((stream == AUDIO_STREAM_BLUETOOTH_SCO) && outputDesc->mRefCount[stream] != 0 && mAudioPolicyVendorControl.isStateInCallOnly(mEngine->getPhoneState()) && index != 0) {
        volumeDb = 0.0f;
     } else if ((stream == AUDIO_STREAM_VOICE_CALL) && outputDesc->mMuteTid[stream] == gettid()) {
        // we just apply frameworks volume setting to zero, not including hal speech volume
        volumeDb = -758.0;// set 0 to audioflinger
     }
     if (outputDesc->setVolume(volumeDb, stream, device, delayMs, force)) {
        if (outputDesc == mPrimaryOutput) {
            ALOGD("checkAndSetVolume stream = %d index = %d mId = %d device = 0x%x(0x%x) delayMs = %d force = %d [%d/0x%x/%d]"
            , stream, index, outputDesc->getId(), device, originDevice, delayMs, force, mVolumeStream, mVolumeDevice, mVolumeIndex);
        } else {
            ALOGVV("checkAndSetVolume stream = %d index = %d mId = %d device = 0x%x(0x%x) delayMs = %d force = %d [%d/0x%x/%d] !P"
            , stream, index, outputDesc->getId(), device, originDevice, delayMs, force, mVolumeStream, mVolumeDevice, mVolumeIndex);
        }
     }
#else
    outputDesc->setVolume(volumeDb, stream, device, delayMs, force);
#endif

#if defined(MTK_AUDIO) && defined(SUPPORT_ANDROID_FM_PLAYER)
        if (stream == AUDIO_STREAM_MUSIC && outputDesc == mPrimaryOutput && (device & (AUDIO_DEVICE_OUT_WIRED_HEADSET | AUDIO_DEVICE_OUT_WIRED_HEADPHONE | AUDIO_DEVICE_OUT_SPEAKER))) {
            for (ssize_t i = 0; i < (ssize_t)mAudioPatches.size(); i++) {
            MTK_ALOGV("%s size %zu/%zu", __FUNCTION__, i, mAudioPatches.size());
            sp<AudioPatch> patchDesc = mAudioPatches.valueAt(i);
            if (isFMDirectMode(patchDesc)) {
                    ALOGD("Do modify audiopatch volume");
                    struct audio_port_config *config;
                    sp<AudioPortConfig> audioPortConfig;
                    sp<DeviceDescriptor> deviceDesc;
                    config = &(patchDesc->mPatch.sinks[0]);
                    bool bOrignalDeviceRemoved = false;
                    if (config->role == AUDIO_PORT_ROLE_SINK) {
                        deviceDesc = mAvailableOutputDevices.getDeviceFromId(config->id);
                    } else {
                        ALOGD("1st deviceDesc NULL");
                        break;
                    }
                    if (deviceDesc == NULL) {
                        bOrignalDeviceRemoved = true;// Headset is removed
                        ALOGD("bOrignalDeviceRemoved Device %x replace %x", device, config->ext.device.type);
                        deviceDesc = mAvailableOutputDevices.getDevice(device, String8(""));
                        if (deviceDesc == NULL) {
                            ALOGD("2nd deviceDesc NULL");
                            break;
                        }
                    }
                    audioPortConfig = deviceDesc;
                    struct audio_port_config newConfig;
                    audioPortConfig->toAudioPortConfig(&newConfig, config);
                    if (bOrignalDeviceRemoved == true)
                        newConfig.ext.device.type = config->ext.device.type;
                    newConfig.config_mask = AUDIO_PORT_CONFIG_GAIN | newConfig.config_mask;
                    newConfig.gain.mode = AUDIO_GAIN_MODE_JOINT | newConfig.gain.mode;
#ifdef MTK_NEW_VOL_CONTROL
                    newConfig.gain.values[0] = index;   // pass volume index directly
#else
                    newConfig.gain.values[0] = -300 * (getStreamMaxLevels(stream) - index);
#endif
                    if ((device != newConfig.ext.device.type || bOrignalDeviceRemoved) && index != 0)// For switch and pop between hp and speaker
                        newConfig.ext.device.type = device; // Device change, Don't un-mute, wait next createAudioPatch
                    mpClientInterface->setAudioPortConfig(&newConfig, delayMs);
                }
            }
        }
#endif

#if defined(MTK_AUDIO) && defined(MTK_AUDIO_GAIN_TABLE)
        if (outputDesc->sharesHwModuleWith(mPrimaryOutput) && originDevice != AUDIO_DEVICE_NONE) {
            checkAndSetGainTableAnalogGain(stream, index, outputDesc, device, delayMs, force);
        }
#elif defined(MTK_AUDIO)
        if (stream == AUDIO_STREAM_VOICE_CALL ||
            stream == AUDIO_STREAM_BLUETOOTH_SCO) {
        float voiceVolume;
        // Force voice volume to max for bluetooth SCO as volume is managed by the headset
        if (stream == AUDIO_STREAM_VOICE_CALL) {
            if ((mEngine->getPhoneState() == AUDIO_MODE_IN_COMMUNICATION)) {
                return NO_ERROR;
            }
            if (mAudioPolicyVendorControl.getCustomVolumeStatus()) {
                voiceVolume = computeCustomVolume(stream, index, device);
            } else {
                voiceVolume = (float)index/(float)mVolumeCurves->getVolumeIndexMax(stream);
            }
        } else {
            voiceVolume = 1.0;
            // it should mute BT if changing to BT from HP/SPK. But hal desn't support 0
            // keep the same value, at least original won't burst before changing device (ALPS02474519)
            if(outputDesc->mMuteTid[stream] == gettid()) {
                voiceVolume = mLastVoiceVolume;
            }
        }
#if defined(MTK_AUDIO_HIERARCHICAL_PARAM_SUPPORT) && !defined(MTK_NEW_VOL_CONTROL)
        if (outputDesc == mPrimaryOutput) {
            // Android N remove outputDesc == mPrimaryOutput condition!!
            // AUDIO_STREAM_VOICE_CALL 0, AUDIO_STREAM_BLUETOOTH_SCO 6
            ALOGD("%s() add volume index Tina, volumeStreamType =0x%x, volumeDevice=0x%x, volumeIndex=0x%x", __FUNCTION__, stream, device, index);
            AudioParameter param = AudioParameter();
            param.addInt(String8("volumeStreamType"), stream);
            param.addInt(String8("volumeDevice"), device);
            param.addInt(String8("volumeIndex"), index);
            mpClientInterface->setParameters(mPrimaryOutput->mIoHandle, param.toString(), delayMs);
        }
#endif
#if defined(EVDO_DT_SUPPORT)
        if (outputDesc == mPrimaryOutput) {
            mpClientInterface->setVoiceVolume(voiceVolume, delayMs);
            ALOGD("SetVoiceVolumeIndex=%d %f,%f", index, voiceVolume, mLastVoiceVolume);
            //set Volume Index ( for external modem)
            AudioParameter param = AudioParameter();
            param.addInt(String8("SetVoiceVolumeIndex"), index);
            mpClientInterface->setParameters(AUDIO_IO_HANDLE_NONE , param.toString(), 0);
            mLastVoiceVolume = voiceVolume;
        }
#else
        if (voiceVolume != mLastVoiceVolume && outputDesc == mPrimaryOutput) {
            mpClientInterface->setVoiceVolume(voiceVolume, delayMs);
            mLastVoiceVolume = voiceVolume;
        }
#endif
    }
#else
    if (stream == AUDIO_STREAM_VOICE_CALL ||
        stream == AUDIO_STREAM_BLUETOOTH_SCO) {
        float voiceVolume;
        // Force voice volume to max for bluetooth SCO as volume is managed by the headset
        if (stream == AUDIO_STREAM_VOICE_CALL) {
            voiceVolume = (float)index/(float)mVolumeCurves->getVolumeIndexMax(stream);
        } else {
            voiceVolume = 1.0;
        }

        if (voiceVolume != mLastVoiceVolume) {
            mpClientInterface->setVoiceVolume(voiceVolume, delayMs);
            mLastVoiceVolume = voiceVolume;
        }
    }
#endif

    return NO_ERROR;
}

void AudioPolicyManager::applyStreamVolumes(const sp<AudioOutputDescriptor>& outputDesc,
                                                audio_devices_t device,
                                                int delayMs,
                                                bool force)
{
    ALOGVV("applyStreamVolumes() for device %08x", device);

    for (int stream = 0; stream < AUDIO_STREAM_FOR_POLICY_CNT; stream++) {
        checkAndSetVolume((audio_stream_type_t)stream,
                          mVolumeCurves->getVolumeIndex((audio_stream_type_t)stream, device),
                          outputDesc,
                          device,
                          delayMs,
                          force);
    }
}

void AudioPolicyManager::setStrategyMute(routing_strategy strategy,
                                             bool on,
                                             const sp<AudioOutputDescriptor>& outputDesc,
                                             int delayMs,
                                             audio_devices_t device)
{
    ALOGVV("setStrategyMute() strategy %d, mute %d, output ID %d",
           strategy, on, outputDesc->getId());
    for (int stream = 0; stream < AUDIO_STREAM_FOR_POLICY_CNT; stream++) {
        if (getStrategy((audio_stream_type_t)stream) == strategy) {
            setStreamMute((audio_stream_type_t)stream, on, outputDesc, delayMs, device);
        }
    }
}

void AudioPolicyManager::setStreamMute(audio_stream_type_t stream,
                                           bool on,
                                           const sp<AudioOutputDescriptor>& outputDesc,
                                           int delayMs,
                                           audio_devices_t device)
{
    if (device == AUDIO_DEVICE_NONE) {
        device = outputDesc->device();
    }

    ALOGVV("setStreamMute() stream %d, mute %d, mMuteCount %d device %04x",
          stream, on, outputDesc->mMuteCount[stream], device);
#ifdef MTK_AUDIO
    // For race-condition between music stream mute/unmute
    AutoMutex _l(outputDesc->mOutputDescStreamLock[stream]);
#endif
    if (on) {
        if (outputDesc->mMuteCount[stream] == 0) {
            if (mVolumeCurves->canBeMuted(stream) &&
                    ((stream != AUDIO_STREAM_ENFORCED_AUDIBLE) ||
                     (mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_SYSTEM) == AUDIO_POLICY_FORCE_NONE))) {
#ifdef MTK_AUDIO    // Because voice stream has no muted tuning value, even index 0 is not muted.
                outputDesc->mMuteTid[stream] = gettid();
                checkAndSetVolume(stream, 0, outputDesc, device, delayMs);
                outputDesc->mMuteTid[stream] = 0;
#else
                checkAndSetVolume(stream, 0, outputDesc, device, delayMs);
#endif
            }
        }
        // increment mMuteCount after calling checkAndSetVolume() so that volume change is not ignored
        outputDesc->mMuteCount[stream]++;
    } else {
        if (outputDesc->mMuteCount[stream] == 0) {
            ALOGV("setStreamMute() unmuting non muted stream!");
            return;
        }
        if (--outputDesc->mMuteCount[stream] == 0) {
#ifdef MTK_AUDIO
            // KH: ALPS00612643. To avoid ringtone leakage during speech call.
            if ( isInCall() && audio_is_low_visibility(stream) ) {
               delayMs = outputDesc->latency();
            }
            outputDesc->mMuteTid[stream] = -gettid();
            checkAndSetVolume(stream,
                              mVolumeCurves->getVolumeIndex(stream, device),
                              outputDesc,
                              device,
                              delayMs);
            outputDesc->mMuteTid[stream] = 0;
#else
            checkAndSetVolume(stream,
                              mVolumeCurves->getVolumeIndex(stream, device),
                              outputDesc,
                              device,
                              delayMs);
#endif
        }
    }
}

void AudioPolicyManager::handleIncallSonification(audio_stream_type_t stream,
                                                      bool starting, bool stateChange)
{
    if(!hasPrimaryOutput()) {
        return;
    }

    // if the stream pertains to sonification strategy and we are in call we must
    // mute the stream if it is low visibility. If it is high visibility, we must play a tone
    // in the device used for phone strategy and play the tone if the selected device does not
    // interfere with the device used for phone strategy
    // if stateChange is true, we are called from setPhoneState() and we must mute or unmute as
    // many times as there are active tracks on the output
    const routing_strategy stream_strategy = getStrategy(stream);
    if ((stream_strategy == STRATEGY_SONIFICATION) ||
            ((stream_strategy == STRATEGY_SONIFICATION_RESPECTFUL))) {
        sp<SwAudioOutputDescriptor> outputDesc = mPrimaryOutput;
        ALOGVV("handleIncallSonification() stream %d starting %d device %x stateChange %d",
                stream, starting, outputDesc->mDevice, stateChange);
        if (outputDesc->mRefCount[stream]) {
            int muteCount = 1;
            if (stateChange) {
                muteCount = outputDesc->mRefCount[stream];
            }
            if (audio_is_low_visibility(stream)) {
                ALOGV("handleIncallSonification() low visibility, muteCount %d", muteCount);
                for (int i = 0; i < muteCount; i++) {
                    setStreamMute(stream, starting, mPrimaryOutput);
                }
            } else {
                ALOGV("handleIncallSonification() high visibility");
                if (outputDesc->device() &
                        getDeviceForStrategy(STRATEGY_PHONE, true /*fromCache*/)) {
                    ALOGV("handleIncallSonification() high visibility muted, muteCount %d", muteCount);
                    for (int i = 0; i < muteCount; i++) {
                        setStreamMute(stream, starting, mPrimaryOutput);
                    }
                }
                if (starting) {
                    mpClientInterface->startTone(AUDIO_POLICY_TONE_IN_CALL_NOTIFICATION,
                                                 AUDIO_STREAM_VOICE_CALL);
                } else {
                    mpClientInterface->stopTone();
                }
            }
        }
    }
}

audio_stream_type_t AudioPolicyManager::streamTypefromAttributesInt(const audio_attributes_t *attr)
{
    // flags to stream type mapping
    if ((attr->flags & AUDIO_FLAG_AUDIBILITY_ENFORCED) == AUDIO_FLAG_AUDIBILITY_ENFORCED) {
        return AUDIO_STREAM_ENFORCED_AUDIBLE;
    }
    if ((attr->flags & AUDIO_FLAG_SCO) == AUDIO_FLAG_SCO) {
        return AUDIO_STREAM_BLUETOOTH_SCO;
    }
    if ((attr->flags & AUDIO_FLAG_BEACON) == AUDIO_FLAG_BEACON) {
        return AUDIO_STREAM_TTS;
    }

    // usage to stream type mapping
    switch (attr->usage) {
    case AUDIO_USAGE_MEDIA:
    case AUDIO_USAGE_GAME:
    case AUDIO_USAGE_ASSISTANCE_NAVIGATION_GUIDANCE:
        return AUDIO_STREAM_MUSIC;
    case AUDIO_USAGE_ASSISTANCE_ACCESSIBILITY:
        return AUDIO_STREAM_ACCESSIBILITY;
    case AUDIO_USAGE_ASSISTANCE_SONIFICATION:
        return AUDIO_STREAM_SYSTEM;
    case AUDIO_USAGE_VOICE_COMMUNICATION:
        return AUDIO_STREAM_VOICE_CALL;

    case AUDIO_USAGE_VOICE_COMMUNICATION_SIGNALLING:
        return AUDIO_STREAM_DTMF;

    case AUDIO_USAGE_ALARM:
        return AUDIO_STREAM_ALARM;
    case AUDIO_USAGE_NOTIFICATION_TELEPHONY_RINGTONE:
        return AUDIO_STREAM_RING;

    case AUDIO_USAGE_NOTIFICATION:
    case AUDIO_USAGE_NOTIFICATION_COMMUNICATION_REQUEST:
    case AUDIO_USAGE_NOTIFICATION_COMMUNICATION_INSTANT:
    case AUDIO_USAGE_NOTIFICATION_COMMUNICATION_DELAYED:
    case AUDIO_USAGE_NOTIFICATION_EVENT:
        return AUDIO_STREAM_NOTIFICATION;
    case AUDIO_USAGE_UNKNOWN:
    default:
        return AUDIO_STREAM_MUSIC;
    }
}

bool AudioPolicyManager::isValidAttributes(const audio_attributes_t *paa)
{
    // has flags that map to a strategy?
    if ((paa->flags & (AUDIO_FLAG_AUDIBILITY_ENFORCED | AUDIO_FLAG_SCO | AUDIO_FLAG_BEACON)) != 0) {
        return true;
    }

    // has known usage?
    switch (paa->usage) {
    case AUDIO_USAGE_UNKNOWN:
    case AUDIO_USAGE_MEDIA:
    case AUDIO_USAGE_VOICE_COMMUNICATION:
    case AUDIO_USAGE_VOICE_COMMUNICATION_SIGNALLING:
    case AUDIO_USAGE_ALARM:
    case AUDIO_USAGE_NOTIFICATION:
    case AUDIO_USAGE_NOTIFICATION_TELEPHONY_RINGTONE:
    case AUDIO_USAGE_NOTIFICATION_COMMUNICATION_REQUEST:
    case AUDIO_USAGE_NOTIFICATION_COMMUNICATION_INSTANT:
    case AUDIO_USAGE_NOTIFICATION_COMMUNICATION_DELAYED:
    case AUDIO_USAGE_NOTIFICATION_EVENT:
    case AUDIO_USAGE_ASSISTANCE_ACCESSIBILITY:
    case AUDIO_USAGE_ASSISTANCE_NAVIGATION_GUIDANCE:
    case AUDIO_USAGE_ASSISTANCE_SONIFICATION:
    case AUDIO_USAGE_GAME:
    case AUDIO_USAGE_VIRTUAL_SOURCE:
        break;
    default:
        return false;
    }
    return true;
}

bool AudioPolicyManager::isStrategyActive(const sp<AudioOutputDescriptor> outputDesc,
                                          routing_strategy strategy, uint32_t inPastMs,
                                          nsecs_t sysTime, bool bShareHwModule) const
{
    if ((sysTime == 0) && (inPastMs != 0)) {
        sysTime = systemTime();
    }
    for (int i = 0; i < (int)AUDIO_STREAM_FOR_POLICY_CNT; i++) {
        if (((getStrategy((audio_stream_type_t)i) == strategy) ||
                (NUM_STRATEGIES == strategy)) &&
                outputDesc->isStreamActive((audio_stream_type_t)i, inPastMs, sysTime)) {
            return true;
        }
#ifdef MTK_AUDIO
        if (bShareHwModule) {
            for (size_t j = 0; j < mOutputs.size(); j++) {
                sp<AudioOutputDescriptor> desc = mOutputs.valueAt(j);
                if (desc != outputDesc && !desc->isDuplicated() && outputDesc->sharesHwModuleWith(desc)) {
                    if (((getStrategy((audio_stream_type_t)i) == strategy) ||
                         (NUM_STRATEGIES == strategy)) &&
                         desc->isStreamActive((audio_stream_type_t)i, inPastMs, sysTime)) {
                        return true;
                    }
                }
            }
        }
#endif
    }
    return false;
}

audio_policy_forced_cfg_t AudioPolicyManager::getForceUse(audio_policy_force_use_t usage)
{
    return mEngine->getForceUse(usage);
}

bool AudioPolicyManager::isInCall()
{
    return isStateInCall(mEngine->getPhoneState());
}

bool AudioPolicyManager::isStateInCall(int state)
{
    return is_state_in_call(state);
}

void AudioPolicyManager::cleanUpForDevice(const sp<DeviceDescriptor>& deviceDesc)
{
    for (ssize_t i = (ssize_t)mAudioSources.size() - 1; i >= 0; i--)  {
        sp<AudioSourceDescriptor> sourceDesc = mAudioSources.valueAt(i);
        if (sourceDesc->mDevice->equals(deviceDesc)) {
            ALOGV("%s releasing audio source %d", __FUNCTION__, sourceDesc->getHandle());
            stopAudioSource(sourceDesc->getHandle());
        }
    }

    for (ssize_t i = (ssize_t)mAudioPatches.size() - 1; i >= 0; i--)  {
        sp<AudioPatch> patchDesc = mAudioPatches.valueAt(i);
        bool release = false;
        for (size_t j = 0; j < patchDesc->mPatch.num_sources && !release; j++)  {
            const struct audio_port_config *source = &patchDesc->mPatch.sources[j];
            if (source->type == AUDIO_PORT_TYPE_DEVICE &&
                    source->ext.device.type == deviceDesc->type()) {
                release = true;
            }
        }
        for (size_t j = 0; j < patchDesc->mPatch.num_sinks && !release; j++)  {
            const struct audio_port_config *sink = &patchDesc->mPatch.sinks[j];
            if (sink->type == AUDIO_PORT_TYPE_DEVICE &&
                    sink->ext.device.type == deviceDesc->type()) {
                release = true;
            }
        }
        if (release) {
            ALOGV("%s releasing patch %u", __FUNCTION__, patchDesc->mHandle);
            releaseAudioPatch(patchDesc->mHandle, patchDesc->mUid);
        }
    }
}

// Modify the list of surround sound formats supported.
void AudioPolicyManager::filterSurroundFormats(FormatVector *formatsPtr) {
    FormatVector &formats = *formatsPtr;
    // TODO Set this based on Config properties.
    const bool alwaysForceAC3 = true;

    audio_policy_forced_cfg_t forceUse = mEngine->getForceUse(
            AUDIO_POLICY_FORCE_FOR_ENCODED_SURROUND);
    ALOGD("%s: forced use = %d", __FUNCTION__, forceUse);

    // Analyze original support for various formats.
    bool supportsAC3 = false;
    bool supportsOtherSurround = false;
    bool supportsIEC61937 = false;
    for (size_t formatIndex = 0; formatIndex < formats.size(); formatIndex++) {
        audio_format_t format = formats[formatIndex];
        switch (format) {
            case AUDIO_FORMAT_AC3:
                supportsAC3 = true;
                break;
            case AUDIO_FORMAT_E_AC3:
            case AUDIO_FORMAT_DTS:
            case AUDIO_FORMAT_DTS_HD:
                supportsOtherSurround = true;
                break;
            case AUDIO_FORMAT_IEC61937:
                supportsIEC61937 = true;
                break;
            default:
                break;
        }
    }

    // Modify formats based on surround preferences.
    // If NEVER, remove support for surround formats.
    if (forceUse == AUDIO_POLICY_FORCE_ENCODED_SURROUND_NEVER) {
        if (supportsAC3 || supportsOtherSurround || supportsIEC61937) {
            // Remove surround sound related formats.
            for (size_t formatIndex = 0; formatIndex < formats.size(); ) {
                audio_format_t format = formats[formatIndex];
                switch(format) {
                    case AUDIO_FORMAT_AC3:
                    case AUDIO_FORMAT_E_AC3:
                    case AUDIO_FORMAT_DTS:
                    case AUDIO_FORMAT_DTS_HD:
                    case AUDIO_FORMAT_IEC61937:
                        formats.removeAt(formatIndex);
                        break;
                    default:
                        formatIndex++; // keep it
                        break;
                }
            }
            supportsAC3 = false;
            supportsOtherSurround = false;
            supportsIEC61937 = false;
        }
    } else { // AUTO or ALWAYS
        // Most TVs support AC3 even if they do not report it in the EDID.
        if ((alwaysForceAC3 || (forceUse == AUDIO_POLICY_FORCE_ENCODED_SURROUND_ALWAYS))
                && !supportsAC3) {
            formats.add(AUDIO_FORMAT_AC3);
            supportsAC3 = true;
        }

        // If ALWAYS, add support for raw surround formats if all are missing.
        // This assumes that if any of these formats are reported by the HAL
        // then the report is valid and should not be modified.
        if ((forceUse == AUDIO_POLICY_FORCE_ENCODED_SURROUND_ALWAYS)
                && !supportsOtherSurround) {
            formats.add(AUDIO_FORMAT_E_AC3);
            formats.add(AUDIO_FORMAT_DTS);
            formats.add(AUDIO_FORMAT_DTS_HD);
            supportsOtherSurround = true;
        }

        // Add support for IEC61937 if any raw surround supported.
        // The HAL could do this but add it here, just in case.
        if ((supportsAC3 || supportsOtherSurround) && !supportsIEC61937) {
            formats.add(AUDIO_FORMAT_IEC61937);
            supportsIEC61937 = true;
        }
    }
}

// Modify the list of channel masks supported.
void AudioPolicyManager::filterSurroundChannelMasks(ChannelsVector *channelMasksPtr) {
    ChannelsVector &channelMasks = *channelMasksPtr;
    audio_policy_forced_cfg_t forceUse = mEngine->getForceUse(
            AUDIO_POLICY_FORCE_FOR_ENCODED_SURROUND);

    // If NEVER, then remove support for channelMasks > stereo.
    if (forceUse == AUDIO_POLICY_FORCE_ENCODED_SURROUND_NEVER) {
        for (size_t maskIndex = 0; maskIndex < channelMasks.size(); ) {
            audio_channel_mask_t channelMask = channelMasks[maskIndex];
            if (channelMask & ~AUDIO_CHANNEL_OUT_STEREO) {
                ALOGI("%s: force NEVER, so remove channelMask 0x%08x", __FUNCTION__, channelMask);
                channelMasks.removeAt(maskIndex);
            } else {
                maskIndex++;
            }
        }
    // If ALWAYS, then make sure we at least support 5.1
    } else if (forceUse == AUDIO_POLICY_FORCE_ENCODED_SURROUND_ALWAYS) {
        bool supports5dot1 = false;
        // Are there any channel masks that can be considered "surround"?
        for (size_t maskIndex = 0; maskIndex < channelMasks.size(); maskIndex++) {
            audio_channel_mask_t channelMask = channelMasks[maskIndex];
            if ((channelMask & AUDIO_CHANNEL_OUT_5POINT1) == AUDIO_CHANNEL_OUT_5POINT1) {
                supports5dot1 = true;
                break;
            }
        }
        // If not then add 5.1 support.
        if (!supports5dot1) {
            channelMasks.add(AUDIO_CHANNEL_OUT_5POINT1);
            ALOGI("%s: force ALWAYS, so adding channelMask for 5.1 surround", __FUNCTION__);
        }
    }
}

void AudioPolicyManager::updateAudioProfiles(audio_devices_t device,
                                             audio_io_handle_t ioHandle,
                                             AudioProfileVector &profiles)
{
    String8 reply;
    char *value;

    // Format MUST be checked first to update the list of AudioProfile
    if (profiles.hasDynamicFormat()) {
        reply = mpClientInterface->getParameters(ioHandle,
                                                 String8(AUDIO_PARAMETER_STREAM_SUP_FORMATS));
        ALOGV("%s: supported formats %s", __FUNCTION__, reply.string());
        AudioParameter repliedParameters(reply);
        if (repliedParameters.get(
                String8(AUDIO_PARAMETER_STREAM_SUP_FORMATS), reply) != NO_ERROR) {
            ALOGE("%s: failed to retrieve format, bailing out", __FUNCTION__);
            return;
        }
        FormatVector formats = formatsFromString(reply.string());
        if (device == AUDIO_DEVICE_OUT_HDMI) {
            filterSurroundFormats(&formats);
        }
        profiles.setFormats(formats);
    }
    const FormatVector &supportedFormats = profiles.getSupportedFormats();

    for (size_t formatIndex = 0; formatIndex < supportedFormats.size(); formatIndex++) {
        audio_format_t format = supportedFormats[formatIndex];
        ChannelsVector channelMasks;
        SampleRateVector samplingRates;
        AudioParameter requestedParameters;
        requestedParameters.addInt(String8(AUDIO_PARAMETER_STREAM_FORMAT), format);

        if (profiles.hasDynamicRateFor(format)) {
            reply = mpClientInterface->getParameters(ioHandle,
                                                     requestedParameters.toString() + ";" +
                                                     AUDIO_PARAMETER_STREAM_SUP_SAMPLING_RATES);
            ALOGV("%s: supported sampling rates %s", __FUNCTION__, reply.string());
            AudioParameter repliedParameters(reply);
            if (repliedParameters.get(
                    String8(AUDIO_PARAMETER_STREAM_SUP_SAMPLING_RATES), reply) == NO_ERROR) {
                samplingRates = samplingRatesFromString(reply.string());
            }
        }
        if (profiles.hasDynamicChannelsFor(format)) {
            reply = mpClientInterface->getParameters(ioHandle,
                                                     requestedParameters.toString() + ";" +
                                                     AUDIO_PARAMETER_STREAM_SUP_CHANNELS);
            ALOGV("%s: supported channel masks %s", __FUNCTION__, reply.string());
            AudioParameter repliedParameters(reply);
            if (repliedParameters.get(
                    String8(AUDIO_PARAMETER_STREAM_SUP_CHANNELS), reply) == NO_ERROR) {
                channelMasks = channelMasksFromString(reply.string());
                if (device == AUDIO_DEVICE_OUT_HDMI) {
                    filterSurroundChannelMasks(&channelMasks);
                }
            }
        }
        profiles.addProfileFromHal(new AudioProfile(format, channelMasks, samplingRates));
    }
}

//MTK_AUDIO
status_t AudioPolicyManager::addAudioPatch(audio_patch_handle_t handle, const sp<AudioPatch>& patch)
{
#if defined(MTK_AUDIO) && defined(SUPPORT_ANDROID_FM_PLAYER)
    ssize_t index = mAudioPatches.indexOfKey(handle);

    if (index >= 0) {
        ALOGW("addAudioPatch() patch %d already in", handle);
        return ALREADY_EXISTS;
    }
    bool bFMeable = false;
    sp<SwAudioOutputDescriptor> outputDesc = mPrimaryOutput;
    if (isFMDirectMode(patch)) {
        if (outputDesc != NULL) {
            ALOGV("audiopatch Music+");
            outputDesc->changeRefCount(AUDIO_STREAM_MUSIC, 1);
            bFMeable = true;
            mFMDirectAudioPatchEnable = true;
            audio_devices_t newDevice = AUDIO_DEVICE_NONE;
            for (size_t j = 0; j < patch->mPatch.num_sinks; j++)  {
                newDevice = newDevice|patch->mPatch.sinks[j].ext.device.type;
            }
            setOutputDevice(outputDesc, newDevice);
        }
    }
#endif

    status_t status = mAudioPatches.addAudioPatch(handle, patch);

#if defined(MTK_AUDIO) && defined(SUPPORT_ANDROID_FM_PLAYER)
    if (bFMeable && status == NO_ERROR) {
        applyStreamVolumes(outputDesc, patch->mPatch.sinks[0].ext.device.type, outputDesc->mLatency*2, true);
    }
#endif
    return status;
}

status_t AudioPolicyManager::removeAudioPatch(audio_patch_handle_t handle)
{
    ssize_t index = mAudioPatches.indexOfKey(handle);
    if (index < 0) {
        ALOGW("removeAudioPatch() patch %d not in", handle);
        return ALREADY_EXISTS;
    }
    ALOGV("removeAudioPatch() handle %d af handle %d", handle,
                      mAudioPatches.valueAt(index)->mAfPatchHandle);
#if defined(MTK_AUDIO) && defined(SUPPORT_ANDROID_FM_PLAYER)
    bool bFMeable = false;
    sp<SwAudioOutputDescriptor> outputDesc = mPrimaryOutput;
    const sp<AudioPatch> patch = mAudioPatches.valueAt(index);
    if (isFMDirectMode(patch)) {
        if (outputDesc != NULL) {
            if (outputDesc->mRefCount[AUDIO_STREAM_MUSIC] > 0) {
                ALOGV("audiopatch Music-");
                outputDesc->changeRefCount(AUDIO_STREAM_MUSIC, -1);
                bFMeable = true;
                mFMDirectAudioPatchEnable = false;
                if (outputDesc->mRefCount[AUDIO_STREAM_MUSIC] == 0) {
                    audio_devices_t newDevice = getNewOutputDevice(outputDesc, false /*fromCache*/, true);
                    setOutputDevice(outputDesc, newDevice, false, outputDesc->latency()*2);
                }
            }
        }
    }
#endif
    return mAudioPatches.removeAudioPatch(handle);
}
//-MTK_AUDIO
#ifdef MTK_AUDIO
#ifdef MTK_NEW_VOL_CONTROL
// ----------------------------------------------------------------------------
// Callbacks
// ----------------------------------------------------------------------------
void gainTableXmlChangedCb(AppHandle *_appHandle, const char *_audioTypeName)
{
    // reload XML file
    AppOps* appOps = appOpsGetInstance();
    if (appOps == NULL) {
        ALOGE("%s(), Error: AppOps == NULL", __FUNCTION__);
        return;
    }

    if (appOps->appHandleReloadAudioType(_appHandle, _audioTypeName) == APP_ERROR) {
        ALOGE("%s(), Reload xml fail!(audioType = %s)", __FUNCTION__, _audioTypeName);
    } else {
        bool isRelated = false;
        if (strcmp(_audioTypeName, PLAY_DIGI_AUDIOTYPE_NAME) == 0 ||
            strcmp(_audioTypeName, SPEECH_VOL_AUDIOTYPE_NAME) == 0 ||
            strcmp(_audioTypeName, VOIP_VOL_AUDIOTYPE_NAME) == 0 ||
            strcmp(_audioTypeName, VOLUME_AUDIOTYPE_NAME) == 0 ||
            strcmp(_audioTypeName, GAIN_MAP_AUDIOTYPE_NAME) == 0)
            isRelated = true;

        if (isRelated) {
            const sp<IAudioPolicyService> &aps = AudioSystem::get_audio_policy_service();
            aps->SetPolicyManagerParameters(POLICY_LOAD_VOLUME, 0, 0, 0);
        }
    }
}
#endif
#ifdef MTK_AUDIO_GAIN_TABLE
float AudioPolicyManager::computeVolume(audio_stream_type_t stream,
                                            int index,
                                            audio_devices_t device,
                                            const sp<AudioOutputDescriptor>& outputDesc)
{
#ifdef MTK_AUDIO // Because we fork android's function, we need to tag where we modify
    float volumeDB;
    if (mAudioPolicyVendorControl.getCustomVolumeStatus()) {
        volumeDB = Volume::AmplToDb(computeGainTableCustomVolume(stream, index, device));
    } else {
        ALOGW("%s,not Customer Volume, Using Android Volume Curve",__FUNCTION__);
        volumeDB = mVolumeCurves->volIndexToDb(stream, Volume::getDeviceCategory(device), index);
    }
    MTK_ALOGV("%s streamtype [%d],index [%d],device [0x%x], volumeDb [%f]", __FUNCTION__, stream, index, device, volumeDB);
#else
    float volumeDB = mVolumeCurves->volIndexToDb(stream, Volume::getDeviceCategory(device), index);
#endif

    // if a headset is connected, apply the following rules to ring tones and notifications
    // to avoid sound level bursts in user's ears:
    // - always attenuate ring tones and notifications volume by 6dB
    // - if music is playing, always limit the volume to current music volume,
    // with a minimum threshold at -36dB so that notification is always perceived.
    const routing_strategy stream_strategy = getStrategy(stream);
#ifdef MTK_AUDIO
    audio_devices_t Streamdevices = getDeviceForStrategy(stream_strategy, true /*fromCache*/);
    if ((device & Streamdevices) && (isStreamActive(AUDIO_STREAM_MUSIC, SONIFICATION_HEADSET_MUSIC_DELAY) ||
            mLimitRingtoneVolume)) {
        device = Streamdevices;
    }
#endif
    if ((device & (AUDIO_DEVICE_OUT_BLUETOOTH_A2DP |
            AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_HEADPHONES |
            AUDIO_DEVICE_OUT_WIRED_HEADSET |
            AUDIO_DEVICE_OUT_WIRED_HEADPHONE)) &&
        ((stream_strategy == STRATEGY_SONIFICATION)
                || (stream_strategy == STRATEGY_SONIFICATION_RESPECTFUL)
                || (stream == AUDIO_STREAM_SYSTEM)
                || ((stream_strategy == STRATEGY_ENFORCED_AUDIBLE) &&
                    (mEngine->getForceUse(AUDIO_POLICY_FORCE_FOR_SYSTEM) == AUDIO_POLICY_FORCE_NONE))) &&
            mVolumeCurves->canBeMuted(stream)) {
        // when the phone is ringing we must consider that music could have been paused just before
        // by the music application and behave as if music was active if the last music track was
        // just stopped
#ifdef MTK_AUDIO
        uint32_t mask = (uint32_t)(device)&(uint32_t)(~AUDIO_DEVICE_OUT_SPEAKER);
        device = (audio_devices_t)(mask) ; // use correct volume index
#endif
        if (isStreamActive(AUDIO_STREAM_MUSIC, SONIFICATION_HEADSET_MUSIC_DELAY) ||
                mLimitRingtoneVolume) {
            volumeDB += SONIFICATION_HEADSET_VOLUME_FACTOR_DB;
            audio_devices_t musicDevice = getDeviceForStrategy(STRATEGY_MEDIA, true /*fromCache*/);
            float musicVolDB = computeVolume(AUDIO_STREAM_MUSIC,
                                             mVolumeCurves->getVolumeIndex(AUDIO_STREAM_MUSIC,
                                                                              musicDevice),
                                             musicDevice, outputDesc);
            float minVolDB = (musicVolDB > SONIFICATION_HEADSET_VOLUME_MIN_DB) ?
                    musicVolDB : SONIFICATION_HEADSET_VOLUME_MIN_DB;
            if (volumeDB > minVolDB) {
                volumeDB = minVolDB;
                ALOGV("computeVolume limiting volume to %f musicVol %f", minVolDB, musicVolDB);
            }
            if (device & (AUDIO_DEVICE_OUT_BLUETOOTH_A2DP |
                    AUDIO_DEVICE_OUT_BLUETOOTH_A2DP_HEADPHONES)) {
                // on A2DP, also ensure notification volume is not too low compared to media when
                // intended to be played
                if ((volumeDB > -96.0f) &&
                        (musicVolDB - SONIFICATION_A2DP_MAX_MEDIA_DIFF_DB > volumeDB)) {
                    ALOGV("computeVolume increasing volume for stream=%d device=0x%X from %f to %f",
                            stream, device,
                            volumeDB, musicVolDB - SONIFICATION_A2DP_MAX_MEDIA_DIFF_DB);
                    volumeDB = musicVolDB - SONIFICATION_A2DP_MAX_MEDIA_DIFF_DB;
                }
            }
        } else if ((Volume::getDeviceForVolume(device) != AUDIO_DEVICE_OUT_SPEAKER) ||
                stream_strategy != STRATEGY_SONIFICATION) {
            volumeDB += SONIFICATION_HEADSET_VOLUME_FACTOR_DB;
        }
    }

#ifdef MTK_AUDIO
    if ((device&AUDIO_DEVICE_OUT_AUX_DIGITAL) &&
        ((stream_strategy == STRATEGY_SONIFICATION)|| (stream_strategy == STRATEGY_SONIFICATION_RESPECTFUL))) {
        ALOGD("AUDIO_DEVICE_OUT_AUX_DIGITAL device = 0x%x stream_strategy = %d", device, stream_strategy);
        if (isStreamActive(AUDIO_STREAM_MUSIC, SONIFICATION_HEADSET_MUSIC_DELAY)) {
            if (volumeDB < SONIFICATION_AUX_DIGITAL_VOLUME_FACTOR_DB) {
                while(volumeDB < SONIFICATION_AUX_DIGITAL_VOLUME_FACTOR_DB)
                    volumeDB = volumeDB + (SONIFICATION_AUX_DIGITAL_VOLUME_FACTOR_DB*(-1));
            }
        }
    }
#endif
    return volumeDB;
}

float AudioPolicyManager::computeGainTableCustomVolume(int stream, int index, audio_devices_t device)
{
    float volume = 1.0;
    device_category deviceCategory = Volume::getDeviceCategory(device);

    if (mAudioPolicyVendorControl.getVoiceReplaceDTMFStatus() && stream == AUDIO_STREAM_DTMF) {
        // normalize new index from 0~15(audio) to 0~7(voice)
        int tempindex = index;
        float DTMFvolInt = (fCUSTOM_VOLUME_MAPPING_STEP * (index - mVolumeCurves->getVolumeIndexMin(AUDIO_STREAM_DTMF))) / (mVolumeCurves->getVolumeIndexMax(AUDIO_STREAM_DTMF) - mVolumeCurves->getVolumeIndexMin(AUDIO_STREAM_DTMF));
        index = (DTMFvolInt*(mVolumeCurves->getVolumeIndexMax(AUDIO_STREAM_VOICE_CALL) - mVolumeCurves->getVolumeIndexMin(AUDIO_STREAM_VOICE_CALL))/ (fCUSTOM_VOLUME_MAPPING_STEP)) + mVolumeCurves->getVolumeIndexMin(AUDIO_STREAM_VOICE_CALL);
        MTK_ALOGVV("volumecheck refine DTMF index [%d] to Voice index [%d]", tempindex, index);
        stream = AUDIO_STREAM_VOICE_CALL;
    }

#ifdef MTK_NEW_VOL_CONTROL
    GAIN_DEVICE gainDevice;
    if (deviceCategory == DEVICE_CATEGORY_SPEAKER) {
        gainDevice = GAIN_DEVICE_SPEAKER;
        if ((device & AUDIO_DEVICE_OUT_WIRED_HEADSET)||
             (device & AUDIO_DEVICE_OUT_WIRED_HEADPHONE))
            gainDevice = GAIN_DEVICE_HSSPK;
    } else if (deviceCategory == DEVICE_CATEGORY_HEADSET) {
        if (device & AUDIO_DEVICE_OUT_WIRED_HEADSET) {
            if (mAudioPolicyVendorControl.getNumOfHeadsetPole() == 5)
                gainDevice = GAIN_DEVICE_HEADSET_5POLE;
            else
                gainDevice = GAIN_DEVICE_HEADSET;
        } else if (device & AUDIO_DEVICE_OUT_WIRED_HEADPHONE) {
            gainDevice = GAIN_DEVICE_HEADPHONE;
        } else {
            gainDevice = GAIN_DEVICE_HEADSET;
        }
    } else if (deviceCategory == DEVICE_CATEGORY_EARPIECE) {
        gainDevice = GAIN_DEVICE_EARPIECE ;
    } else if (deviceCategory == DEVICE_CATEGORY_EXT_MEDIA) {
        gainDevice = GAIN_DEVICE_SPEAKER;
    } else {
        gainDevice = GAIN_DEVICE_SPEAKER;
    }

    ALOG_ASSERT(index >= 0 && index < GAIN_VOL_INDEX_SIZE, "invalid index");
    uint8_t customGain = KeyvolumeStep - mGainTable.streamGain[stream][gainDevice][index].digital;
#else
    AUDIO_GAIN_TABLE_STRUCT *gainTable = &mCustomGainTableVolume;
    STREAM_GAIN *streamGain = &gainTable->voiceCall + stream;
    output_gain_device gainDevice = GAIN_OUTPUT_SPEAKER;
    if (deviceCategory == DEVICE_CATEGORY_SPEAKER) {
        gainDevice = GAIN_OUTPUT_SPEAKER;
    } else if (deviceCategory == DEVICE_CATEGORY_HEADSET) {
        gainDevice = GAIN_OUTPUT_HEADSET;
    } else if (deviceCategory == DEVICE_CATEGORY_EARPIECE) {
        gainDevice = GAIN_OUTPUT_EARPIECE ;
    }
    uint8_t customGain = KeyvolumeStep - streamGain->stream[gainDevice].digital[index];
#endif
    volume = linearToLog(customGain);
    MTK_ALOGV("-computeGainTableCustomVolume customGain 0x%x, volume %f stream %d, index %d, device 0x%x [%d/0x%x/%d]", customGain, volume, stream, index, device, mVolumeStream, mVolumeDevice, mVolumeIndex);
    return volume;
}

int AudioPolicyManager::selectGainTableActiveStream(int requestStream)
{
    int activeStream = requestStream;

    if (isInCall()) {
        if (requestStream == AUDIO_STREAM_BLUETOOTH_SCO)
            activeStream = AUDIO_STREAM_BLUETOOTH_SCO;
        else if (requestStream == AUDIO_STREAM_VOICE_CALL)
            activeStream = AUDIO_STREAM_VOICE_CALL;
        else
            activeStream = AUDIO_STREAM_DEFAULT;
    }
    else if (isStreamActive(AUDIO_STREAM_BLUETOOTH_SCO)) {
        activeStream = AUDIO_STREAM_BLUETOOTH_SCO;
    }
    else if (isStreamActive(AUDIO_STREAM_VOICE_CALL)) {
        activeStream = AUDIO_STREAM_VOICE_CALL;
    }
    else if (isStreamActive(AUDIO_STREAM_ACCESSIBILITY)) {
        activeStream = AUDIO_STREAM_ACCESSIBILITY;
    }
    else if (isStreamActive(AUDIO_STREAM_RING)) {
        activeStream = AUDIO_STREAM_RING;
    }
    else if (isStreamActive(AUDIO_STREAM_ALARM)) {
        activeStream = AUDIO_STREAM_ALARM;
    }
    else if (isStreamActive(AUDIO_STREAM_NOTIFICATION)) {
        activeStream = AUDIO_STREAM_NOTIFICATION;
    }
    else if (isStreamActive(AUDIO_STREAM_ENFORCED_AUDIBLE)) {
        activeStream = AUDIO_STREAM_ENFORCED_AUDIBLE;
    }
    else if (isStreamActive(AUDIO_STREAM_MUSIC)) {
      activeStream = AUDIO_STREAM_MUSIC;
    }
    else if (isStreamActive(AUDIO_STREAM_TTS)) {
        activeStream = AUDIO_STREAM_TTS;
    }
    else if (isStreamActive(AUDIO_STREAM_SYSTEM)) {
        activeStream = AUDIO_STREAM_SYSTEM;
    }
    else if (isStreamActive(AUDIO_STREAM_DTMF)) {
        activeStream = AUDIO_STREAM_DTMF;
    }
    else {
        activeStream = AUDIO_STREAM_DEFAULT;
    }
    return activeStream;
}

status_t AudioPolicyManager::checkAndSetGainTableAnalogGain(int stream, int index, const sp<AudioOutputDescriptor>& outputDesc,audio_devices_t device,
                                                           int delayMs, bool force)
{
    // AudioOutputDescriptor *outputDesc = mOutputs.valueFor(output);
    // sp<AudioOutputDescriptor> outputDesc = mOutputs.valueFor(mPrimaryOutput);
    // sp<SwAudioOutputDescriptor> outputDesc = mPrimaryOutput;
    int activeStream = selectGainTableActiveStream(stream);
    if (activeStream <= AUDIO_STREAM_DEFAULT) {
        return NO_ERROR;
    }
    // audio_devices_t curDevice  = getDeviceForVolume(device);

    if (activeStream != stream) {
        index = mVolumeCurves->getVolumeIndex((audio_stream_type_t)activeStream, device); // mStreams.valueFor((audio_stream_type_t)activeStream).getVolumeIndex(device);
    }

    if ((activeStream == AUDIO_STREAM_VOICE_CALL || activeStream == AUDIO_STREAM_BLUETOOTH_SCO) &&
        outputDesc != mPrimaryOutput) {
        // in voice, set to primary only once, skip others
        return NO_ERROR;
    }

    if (outputDesc->mMuteCount[activeStream] != 0) {    //ALPS02455793. If music stream muted, don't pass music stream volume
        ALOGVV("checkAndSetGainTableAnalogGain() active %d stream %d muted count %d",
              activeStream, stream, outputDesc->mMuteCount[activeStream]);
        return NO_ERROR;
    }

    if (mVolumeStream != activeStream || mVolumeIndex != index || mVolumeDevice != device || force ) {
        ALOGD("computeAndSetAnalogGain stream %d, device 0x%x, index %d", activeStream, device, index);
        mVolumeStream = activeStream;
        mVolumeIndex  = index ;
        mVolumeDevice = device;
        AudioParameter param = AudioParameter();
        param.addInt(String8("volumeStreamType"), activeStream);
        param.addInt(String8("volumeDevice"), device);
        param.addInt(String8("volumeIndex"), index);
        mpClientInterface->setParameters(mPrimaryOutput->mIoHandle, param.toString(),delayMs);
    }
    return NO_ERROR;
}
#else
float AudioPolicyManager::computeVolume(audio_stream_type_t stream __unused,
                                            int index __unused,
                                            audio_devices_t device __unused,
                                            const sp<AudioOutputDescriptor>& outputDesc __unused)
{
    return 0.0;
}

float AudioPolicyManager::computeGainTableCustomVolume(int stream __unused, int index __unused, audio_devices_t device __unused)
{
    return 0.0;
}
int AudioPolicyManager::selectGainTableActiveStream(int requestStream __unused)
{
    return AUDIO_STREAM_DEFAULT;
}
status_t AudioPolicyManager::checkAndSetGainTableAnalogGain(int stream __unused, int index __unused, const sp<AudioOutputDescriptor>& outputDesc __unused,audio_devices_t device __unused,
                                                           int delayMs __unused, bool force __unused)
{
    return INVALID_OPERATION;
}
#endif

uint32_t AudioPolicyManager::GetSampleRateIndex(uint32_t sampleRate)
{
    if (sampleRate >=  OUTPUT_RATE_192) {
        return OUTPUT_RATE_192_INDEX;
    } else if (sampleRate >= OUTPUT_RATE_176_4) {
        return OUTPUT_RATE_176_4_INDEX;
    } else if (sampleRate >= OUTPUT_RATE_96) {
        return OUTPUT_RATE_96_INDEX ;
    } else if (sampleRate >= OUTPUT_RATE_88_2) {
        return OUTPUT_RATE_88_2_INDEX;
    } else if (sampleRate >= OUTPUT_RATE_48) {
        return OUTPUT_RATE_48_INDEX;
    } else {
        return OUTPUT_RATE_44_1_INDEX;
    }
    return OUTPUT_RATE_44_1_INDEX;
}

uint32_t AudioPolicyManager::GetSampleRateCount()
{
    uint32_t index = 0;
    for (int i = 0 ; i < OUTPUT_NUM_RATE_INDEX ; i++) {
        index += SampleRateArray[i];
    }
    MTK_ALOGV("%s index %d", __FUNCTION__, index);
    return index;
}

bool AudioPolicyManager::CheckFirstActive(void)
{
    uint32_t index = 0;

    for (int i = 0 ; i < OUTPUT_NUM_RATE_INDEX ; i++) {
        index += SampleRateArray[i];
    }

    if (index == 1) {
        return true;
    }

    return false;
}

bool AudioPolicyManager::CheckStreamActive(void)
{
    for (int i = 0 ; i < OUTPUT_NUM_RATE_INDEX ; i++) {
        if (SampleRateArray[i] != 0) {
            return true;
        }
    }

    return false;
}

void AudioPolicyManager::DumpSampleRateArray(void)
{
//#define DEBUG_SAMPLERATE_ARRAY
#ifdef DEBUG_SAMPLERATE_ARRAY
    for (int i = 0 ; i < OUTPUT_NUM_RATE_INDEX ; i++) {
        if (SampleRateArray[i] > 0) {
            ALOGD("SampleRateArray[%d] = %d ", i, SampleRateArray[i]);
        }
    }
#endif
}

bool AudioPolicyManager::SetFMIndirectMode(uint32_t SampleRate)
{
    if (SampleRate > 48000) {
        ALOGD("SetFMIndirectMode SampleRate = %d ( > 48000)", SampleRate);
        AudioParameter param;
        param.addInt(String8(AudioParameter::keyFmDirectControl), 0);
        mpClientInterface->setParameters(mPrimaryOutput->mIoHandle, param.toString(),0); //Todo if fm_service is ready
        return true;
    } else {
        return false;
    }
}

bool AudioPolicyManager::ReleaseFMIndirectMode(uint32_t SampleRate)
{
    if (SampleRate > 48000) {
        ALOGD("ReleaseFMIndirectMode ");
    }
    AudioParameter param;
    param.addInt(String8(AudioParameter::keyFmDirectControl), 1);
    mpClientInterface->setParameters(mPrimaryOutput->mIoHandle, param.toString(),0); //Todo if fm_service is ready
    return true;
}
uint32_t AudioPolicyManager::GetFirstTrackSampleRate()
{
    uint32_t SampleRate = 0;
    for (int i = 0 ; i < OUTPUT_NUM_RATE_INDEX ; i++) {
        if (SampleRateArray[i]) {
            switch (i) {
                case OUTPUT_RATE_44_1_INDEX:
                    SampleRate = OUTPUT_RATE_44_1;
                    break;
                case OUTPUT_RATE_48_INDEX:
                    SampleRate = OUTPUT_RATE_48;
                    break;
                case OUTPUT_RATE_88_2_INDEX:
                    SampleRate = OUTPUT_RATE_88_2;
                    break;
                case OUTPUT_RATE_96_INDEX:
                    SampleRate = OUTPUT_RATE_96;
                    break;
                case OUTPUT_RATE_176_4_INDEX:
                    SampleRate = OUTPUT_RATE_176_4;
                    break;
                case OUTPUT_RATE_192_INDEX:
                    SampleRate = OUTPUT_RATE_192;
                    break;
            }
            break;
        }
    }
    return SampleRate;
}

bool AudioPolicyManager::PrimaryDeviceSupportSampleRate(uint32_t SampleRate __unused)
{
    return true;
}


bool AudioPolicyManager::PrimarySupportSampleRate(uint32_t SampleRate)
{
    //ssize_t index = mOutputs.indexOfKey(mPrimaryOutput);
    //sp<AudioOutputDescriptor> outputDesc = mOutputs.valueAt(index);
    sp<SwAudioOutputDescriptor>  outputDesc = mPrimaryOutput;//->mIoHandle;
    const sp<IOProfile> profile = outputDesc->mProfile;
#if 0
    for (size_t i = 0; i < profile->mSamplingRates.size(); i++) {
        MTK_ALOGV("PrimarySupportSampleRate mSamplingRates[%zu] = %d", i, profile->mSamplingRates[i]);
        if (profile->mSamplingRates[i] == SampleRate) {
            return true;
        }
    }
#endif
    if (profile->isSamplingRateSupport(SampleRate) == true) {
        MTK_ALOGV("PrimarySupportSampleRate Support sampling rate = %d", SampleRate);
        return true;
    } else {
        ALOGD("PrimarySupportSampleRate doesn't support SampleRate = %d", SampleRate);
        return false;
    }
}

status_t AudioPolicyManager::PolicyFirstStart(audio_stream_type_t stream , uint32_t sampleRate)
{
    uint32_t SampleRate = 0;
    ALOGD("PolicyFirstStart stream = %d sampleRate = %d, CheckFirstActive() %d mSampleRate_Policy = %d", stream, sampleRate, CheckFirstActive(), mSampleRate_Policy);
    if (CheckFirstActive() == true) {
        SampleRate = GetFirstTrackSampleRate();
        if (PrimarySupportSampleRate(SampleRate) && (PolicySampleRate != SampleRate)) {
            mSampleRateFocusLock.lock();
            PolicySampleRate = mSampleRateFocus ? mSampleRateFocus : SampleRate;
            mSampleRateFocusLock.unlock();
            SampleRate = PolicySampleRate;
            AudioParameter param = AudioParameter();
            param.addInt(String8(AudioParameter::keySamplingRate), (int)PolicySampleRate);
            mpClientInterface->setParameters(mPrimaryOutput->mIoHandle, param.toString());

            audio_io_handle_t fastOutput = getPrimaryFastOutput();
            if(fastOutput) {
                mpClientInterface->setParameters(fastOutput, param.toString());
            }
        }
        mSampleRateFocusLock.lock();
        mSampleRateFocus = 0;
        mSampleRateFocusLock.unlock();
        SetFMIndirectMode(SampleRate);
    }
    return NO_ERROR;
}

int AudioPolicyManager::PolicyPropertyCheck(void)
{
    int result=0;
    char value[PROPERTY_VALUE_MAX];
    property_get(PROPERTY_KEY_POLICY_MODE, value, "0");
    result = atoi(value);
    return result;
}


status_t AudioPolicyManager::PolicyRestore(void)
{
    return NO_ERROR;// ?
#if 0
    if (mSampleRate_Policy !=SampleRate_Do_nothing) {
        PolicySampleRate = 44100;
        AudioParameter param = AudioParameter();
        param.addInt(String8(AudioParameter::keySamplingRate), (int)PolicySampleRate);
        mpClientInterface->setParameters(mPrimaryOutput->mIoHandle, param.toString(),1000);
        mpClientInterface->setParameters(getPrimaryFastOutput(), param.toString(),1000);
    }
    return NO_ERROR;
#endif
}

status_t AudioPolicyManager::PolicyForceReplace(audio_stream_type_t stream, uint32_t sampleRate)
{
    // only take place when music stream, FM is an exception for samplerate change
    ALOGD("PolicyForceReplace stream = %d sampleRate = %d ", stream, sampleRate);
    if (sampleRate <= OUTPUT_RATE_44_1) {
        sampleRate = OUTPUT_RATE_44_1;
    }

    if (stream == AUDIO_STREAM_MUSIC && !isFMActive()) {
        if (PrimarySupportSampleRate(sampleRate) && (PolicySampleRate != sampleRate)) {
            mSampleRateFocusLock.lock();
            PolicySampleRate = mSampleRateFocus ? mSampleRateFocus : sampleRate;
            mSampleRateFocusLock.unlock();
            sampleRate = PolicySampleRate;
            AudioParameter param = AudioParameter();
            param.addInt(String8(AudioParameter::keySamplingRate), (int)PolicySampleRate);
            mpClientInterface->setParameters(mPrimaryOutput->mIoHandle, param.toString());

            audio_io_handle_t fastOutput = getPrimaryFastOutput();
            if(fastOutput) {
                mpClientInterface->setParameters(fastOutput, param.toString());
            }
        }
        SetFMIndirectMode(sampleRate);
    }
    return NO_ERROR;
}

// function to make deision for audio
status_t AudioPolicyManager::SampleRatePolicy(audio_stream_type_t stream, uint32_t sampleRate)
{
    // and then default policy
    if (mSampleRate_Policy == SampleRate_First_Start) {
        PolicyFirstStart(stream, sampleRate);
    }
    else if (mSampleRate_Policy == SampleRate_ForceReplace) {
        PolicyForceReplace(stream, sampleRate);
    }
    else if (mSampleRate_Policy == SampleRate_Do_nothing) {
        // do nothing
    }
    return NO_ERROR;
}


status_t AudioPolicyManager::AddSampleRateArray(audio_stream_type_t stream __unused, uint32_t sampleRate)
{
    uint32_t index = 0;
    index = GetSampleRateIndex(sampleRate);
    MTK_ALOGV("AddSampleRateArray index = %d sampleRate = %d", index, sampleRate);
    SampleRateArray[index]++;
    DumpSampleRateArray();
    return NO_ERROR;
}

status_t AudioPolicyManager::RemoveSampleRateArray(audio_stream_type_t stream __unused, uint32_t sampleRate)
{
    uint32_t index = 0;
    index = GetSampleRateIndex(sampleRate);
    SampleRateArray[index]--;
    DumpSampleRateArray();
    return NO_ERROR;
}

status_t AudioPolicyManager::SampleRateRequestFocus(audio_io_handle_t output,
                                                      audio_stream_type_t stream,
                                                      int *samplerate)
{
    MTK_ALOGV("+%s, output = %d getPrimaryFastOutput = %d stream = %d samplerate = %d mSampleRateFocus ="
          "%d PolicySampleRate = %d GetSampleRateCount() = %d mSampleRateFocusCount = %d", __FUNCTION__,
          output, getPrimaryFastOutput(), stream, *samplerate, mSampleRateFocus, PolicySampleRate,
          GetSampleRateCount(), mSampleRateFocusCount);

    mSampleRateFocusLock.lock();
    if(output == mPrimaryOutput->mIoHandle || output == getPrimaryFastOutput()) {
        if(!mSampleRateFocus /*&& !mSampleRateFocusCount*/ && !GetSampleRateCount() &&
            PrimarySupportSampleRate(*samplerate)) {
            mSampleRateFocus = *samplerate;
        } else {
            *samplerate = mSampleRateFocus ? mSampleRateFocus : PolicySampleRate;
        }
        mSampleRateFocusCount ++;
    }
    mSampleRateFocusLock.unlock();

    ALOGVV("-%s, samplerate %d, mSampleRateFocus %d", __FUNCTION__, *samplerate, mSampleRateFocus);
    return NO_ERROR;
}

status_t AudioPolicyManager::SampleRateUnrequestFocus(audio_io_handle_t output,
                                                      audio_stream_type_t stream,
                                                      int samplerate)
{
    MTK_ALOGV("+%s, output = %d getPrimaryFastOutput = %d stream = %d samplerate = %d mSampleRateFocus ="
          "%d PolicySampleRate = %d GetSampleRateCount() = %d mSampleRateFocusCount = %d", __FUNCTION__,
          output, getPrimaryFastOutput(), stream, samplerate, mSampleRateFocus, PolicySampleRate,
          GetSampleRateCount(), mSampleRateFocusCount);

    mSampleRateFocusLock.lock();
    if(output == mPrimaryOutput->mIoHandle || output == getPrimaryFastOutput()) {
        mSampleRateFocusCount --;
        if(!mSampleRateFocusCount) {
            mSampleRateFocus = 0;
        }
    }
    mSampleRateFocusLock.unlock();

    ALOGVV("-%s, samplerate %d, mSampleRateFocus %d", __FUNCTION__, samplerate, mSampleRateFocus);
    return NO_ERROR;
}

status_t AudioPolicyManager::StartOutputSamplerate(audio_io_handle_t output,
                                                      audio_stream_type_t stream,
                                                      audio_session_t session , int samplerate)
{
    ALOGD("StartOutputSamplerate +output = %d +stream = %d session = %d +samplerate=%d mPrimaryOutput= %d", output, stream, session, samplerate, mPrimaryOutput->mIoHandle);
    status_t startOutputValue = startOutput(output, stream, session);
    if ((output == mPrimaryOutput->mIoHandle || output == getPrimaryFastOutput()) && startOutputValue == NO_ERROR) {
        AddSampleRateArray(stream, (uint32_t) samplerate);
        SampleRatePolicy(stream, (uint32_t) samplerate);
    }
    return startOutputValue;
}

status_t AudioPolicyManager::StopOutputSamplerate(audio_io_handle_t output,
                                                     audio_stream_type_t stream,
                                                     audio_session_t session , int samplerate)
{
    ALOGD("StopOutputSampletate -output = %d -stream = %d session = %d -samplerate=%d mPrimaryOutput = %d", output, stream, session, samplerate, mPrimaryOutput->mIoHandle);

    ssize_t index = mOutputs.indexOfKey(output);
    if (index < 0) {
        ALOGW("stopOutput() unknown output %d", output);
        return BAD_VALUE;
    }
    sp<SwAudioOutputDescriptor> outputDesc = mOutputs.valueAt(index);

    if ((outputDesc->mRefCount[stream] > 0) && ((output == mPrimaryOutput->mIoHandle) || output == getPrimaryFastOutput())) {
        RemoveSampleRateArray(stream, (uint32_t) samplerate);
        if (CheckStreamActive() == false) {
            ReleaseFMIndirectMode((uint32_t)samplerate);
            PolicyRestore();
        }
    }
    return stopOutput(output, stream, session);
}

void AudioPolicyManager::InitSamplerateArray(uint32_t init_sample_rate)
{
    ALOGD("InitSamplerateArray");
    for (int i = 0 ; i < OUTPUT_NUM_RATE_INDEX ; i++) {
        SampleRateArray[i] = 0;
    }
    mSampleRate_Policy = SampleRate_First_Start;
    if (init_sample_rate)
        PolicySampleRate = init_sample_rate;
    else
        PolicySampleRate = 44100;
    mSampleRateFocus = mSampleRateFocusCount = 0;

    int ret = PolicyPropertyCheck();
    if ( ret ) {
        mSampleRate_Policy = ret;
        ALOGD("PolicyPropertyCheck mSampleRate_Policy = %d", mSampleRate_Policy);
    }
}

status_t AudioPolicyManager::SetPolicyManagerParameters(int par1,int par2 ,int par3,int par4)
{
    audio_devices_t primaryOutDevices = mPrimaryOutput->device();
    audio_devices_t curDevice =Volume::getDeviceForVolume(mPrimaryOutput->device());
    ALOGD("SetPolicyManagerParameters par1 = %d par2 = %d par3 = %d par4 = %d curDevice = 0x%x", par1, par2, par3, par4, curDevice);
    status_t volStatus;
    switch(par1) {
#ifdef MTK_CROSSMOUNT_SUPPORT
        case POLICY_SET_CROSSMOUNT_LOCAL_PLAYBACK:{
    #if 1
            AudioParameter param;
            if (par2 == 1) {
                mAudioPolicyVendorControl.setCrossMountLocalPlayback(true);
                param.addInt(String8("CrossMountLocalPlaybackInPrimary"), 1);
            } else {
                mAudioPolicyVendorControl.setCrossMountLocalPlayback(false);
                param.addInt(String8("CrossMountLocalPlaybackInPrimary"), 0);
            }
            setForceUse(AUDIO_POLICY_FORCE_USE_CNT,AUDIO_POLICY_FORCE_NONE);
#ifdef MTK_CROSSMOUNT_MULTI_CH_SUPPORT
            mpClientInterface->setParameters (mPrimaryOutput->mIoHandle, param.toString(), 0);
#endif
    #else
            if (par2 == 0) {
                setStreamMute(AUDIO_STREAM_MUSIC, true, mPrimaryOutput);
                AudioParameter param;
                param.addInt(String8("MutePrimaryMusicStream"), 1);
                mpClientInterface->setParameters (mPrimaryOutput , param.toString(), 0);
            } else {
                setStreamMute(AUDIO_STREAM_MUSIC, false, mPrimaryOutput);
                AudioParameter param;
                param.addInt(String8("MutePrimaryMusicStream"), 0);
                mpClientInterface->setParameters (mPrimaryOutput , param.toString(), 0);
            }
    #endif
            break;
        }
        case POLICY_SET_CROSSMOUNT_MIC_LOCAL_PLAYBACK:{
            bool bprevCrossMountMicLocalPlayback = mAudioPolicyVendorControl.getCrossMountMicLocalPlayback();
            mAudioPolicyVendorControl.setCrossMountMicLocalPlayback(par2);
            if (mAvailableOutputDevices.getDevice(AUDIO_DEVICE_OUT_REMOTE_SUBMIX, String8("0")) == 0) {
                if (bprevCrossMountMicLocalPlayback && !mAudioPolicyVendorControl.getCrossMountMicLocalPlayback()) {
                    ALOGD("CrossMountMIC-Disable Record");
                    CrossMountSink::notifyMicStatus(false);
                } else if (!bprevCrossMountMicLocalPlayback && mAudioPolicyVendorControl.getCrossMountMicLocalPlayback()) {
                    ALOGD("CrossMountMIC-Enable Record");
                    CrossMountSink::notifyMicStatus(true);
                }
            }
            setForceUse(AUDIO_POLICY_FORCE_USE_CNT,AUDIO_POLICY_FORCE_NONE);
            break;
        }
#endif
#ifdef MTK_HDMI_MULTI_CHANNEL_SUPPORT
        case POLICY_SET_HDMI_CHANNEL_SUPPORT:{
            mAudioPolicyVendorControl.setHDMI_ChannelCount(par2);
            setForceUse(AUDIO_POLICY_FORCE_USE_CNT,AUDIO_POLICY_FORCE_NONE);
            break;
        }
#endif
        case POLICY_SET_FM_TX_ENABLE:{
            ALOGD("Set POLICY_SET_FM_TX_ENABLE [%d] => [%d]", mAudioPolicyVendorControl.getFMTxStatus(), par2);
            if (par2) {
                mAudioPolicyVendorControl.setFMTxStatus(true);
            } else {
                mAudioPolicyVendorControl.setFMTxStatus(false);
            }
            setForceUse(AUDIO_POLICY_FORCE_USE_CNT,AUDIO_POLICY_FORCE_NONE);
            break;
        }
        case POLICY_SET_FM_RX_FORCE_DISABLE_TX:{
            ALOGD("Set POLICY_SET_FM_RX_FORCE_DISABLE_TX [%d] => [%d]", mAudioPolicyVendorControl.getFMTxStatus(), par2);
            if (par2) {
                mAudioPolicyVendorControl.setFMTxStatus(false);
            } else {
                mAudioPolicyVendorControl.setFMTxStatus(true);
            }
            setForceUse(AUDIO_POLICY_FORCE_USE_CNT,AUDIO_POLICY_FORCE_NONE);
            break;
        }
        case POLICY_SET_FM_PRESTOP:{
#ifdef SUPPORT_ANDROID_FM_PLAYER//TK only, Don't enable it for BSP
             if (par2) {
                ALOGD("mute for FM app");
                setStreamMute(AUDIO_STREAM_MUSIC, true, mPrimaryOutput);
            } else {
                ALOGD("unmute for FM app");
                setStreamMute(AUDIO_STREAM_MUSIC, false, mPrimaryOutput);
            }
#endif
            break;
        }
        case POLICY_SET_A2DP_FORCE_IGNORE:{
            ALOGD("Set bA2DPForeceIgnore [%d] => [%d]", mAudioPolicyVendorControl.getA2DPForeceIgnoreStatus(), par2);
            if (par2) {
                mAudioPolicyVendorControl.setA2DPForeceIgnoreStatus(true);
            } else {
                mAudioPolicyVendorControl.setA2DPForeceIgnoreStatus(false);
            }
            setForceUse(AUDIO_POLICY_FORCE_USE_CNT,AUDIO_POLICY_FORCE_NONE);
            break;
        }
        case POLICY_LOAD_VOLUME:{
#ifdef MTK_NEW_VOL_CONTROL
            ALOGD("%s(), reload GainTable param", __FUNCTION__);
            if (mpClientInterface->getCustomAudioVolume(&mGainTable)) {
                ALOGE("error, load GainTable failed!!");
            }
#else
            LoadCustomVolume();
#endif
            for(int i =0; i<AUDIO_STREAM_CNT;i++) {
                if (i == AUDIO_STREAM_PATCH) {
                    continue;
                }
                volStatus =checkAndSetVolume((audio_stream_type_t)i, mVolumeCurves->getVolumeIndex((audio_stream_type_t) i, primaryOutDevices), mPrimaryOutput,primaryOutDevices,50,true);
            }
            break;
         }
        case POLICY_SET_TTY_MODE: {
            ALOGD("%s, SetTtyMode=%d", __FUNCTION__, par2);
            mTty_Ctm = (tty_mode_t)par2;
            break;
        }
        case POLICY_SET_NUM_HS_POLE: {
            mAudioPolicyVendorControl.setNumOfHeadsetPole(par2);
            break;
        }
        default:
            break;
    }

    return NO_ERROR;
}

float AudioPolicyManager::linearToLog(int volume)
{
#ifdef MTK_AUDIO_GAIN_TABLE
    return volume ? exp(float(KeyvolumeStep - volume) * KeydBConvert) : 0;
#else
    return volume ? exp(float(fCUSTOM_VOLUME_MAPPING_STEP - volume) * fBConvert) : 0;
#endif
}

int AudioPolicyManager::logToLinear(float volume)
{
#ifdef MTK_AUDIO_GAIN_TABLE
    return volume ? KeyvolumeStep - int(KeydBConvertInverse * log(volume) + 0.5) : 0;
#else
    return volume ? fCUSTOM_VOLUME_MAPPING_STEP - int(fBConvertInverse * log(volume) + 0.5) : 0;
#endif
}

int AudioPolicyManager::mapVol(float &vol, float unitstep)
{
    int index = (vol+0.5)/unitstep;
    vol -= (index*unitstep);
    return index;
}

int AudioPolicyManager::mapping_Voice_vol(float &vol, float unitstep)
{
    #define ROUNDING_NUM (1)

    if (vol < unitstep) {
        return 1;
    }
    if (vol < (unitstep*2 + ROUNDING_NUM)) {
        vol -= unitstep;
        return 2;
    } else if (vol < (unitstep*3 + ROUNDING_NUM)) {
        vol -= unitstep*2;
        return 3;
    } else if (vol < (unitstep*4 + ROUNDING_NUM)) {
        vol -= unitstep*3;
        return 4;
    } else if (vol < (unitstep*5 + ROUNDING_NUM)) {
        vol -= unitstep*4;
        return 5;
    } else if (vol < (unitstep*6 + ROUNDING_NUM)) {
        vol -= unitstep*5;
        return 6;
    } else if (vol < (unitstep*7 + ROUNDING_NUM)) {
        vol -= unitstep*6;
        return 7;
    } else {
        ALOGW("vole = %f unitstep = %f", vol, unitstep);
        return 0;
    }
}


int AudioPolicyManager::getStreamMaxLevels(int stream)
{
    return (int) mAudioCustVolumeTable.audiovolume_level[stream];
}

// this function will map vol 0~100 , base on customvolume map to 0~255 , and do linear calculation to set mastervolume
float AudioPolicyManager::mapVoltoCustomVol(unsigned char array[], int volmin, int volmax,float &vol , int stream)
{
    MTK_ALOGV("+MapVoltoCustomVol vol = %f stream = %d volmin = %d volmax = %d", vol, stream, volmin, volmax);
    CustomVolumeType vol_stream = (CustomVolumeType) stream;
    audio_stream_type_t audio_stream = (audio_stream_type_t) stream;

    if (vol_stream == CUSTOM_VOL_TYPE_VOICE_CALL || vol_stream == CUSTOM_VOL_TYPE_SIP) {
        return mapVoiceVoltoCustomVol(array,volmin,volmax,vol,stream);
    } else if (vol_stream >= CUSTOM_NUM_OF_VOL_TYPE || vol_stream < CUSTOM_VOL_TYPE_VOICE_CALL) {
        ALOGE("%s %d Error : stream = %d", __FUNCTION__, __LINE__, stream);
        audio_stream = AUDIO_STREAM_MUSIC;
        vol_stream = CUSTOM_VOL_TYPE_MUSIC;
    }

    float volume =0.0;
    if (vol == 0) {
        volume = vol;
        return 0;
    } else {    // map volume value to custom volume
        int dMaxLevels = getStreamMaxLevels(vol_stream);
        int streamDescmIndexMax = mVolumeCurves->getVolumeIndexMax(audio_stream);// streamDesc.getVolumeIndexMax();
        if (dMaxLevels <= 0) {
            ALOGE("%s %d Error : dMaxLevels = %d", __FUNCTION__, __LINE__, dMaxLevels);
            dMaxLevels = 1;
        }
        if (streamDescmIndexMax <= 0) {
            ALOGE("%s %d Error : streamDescmIndexMax = %d", __FUNCTION__, __LINE__, streamDescmIndexMax);
            streamDescmIndexMax = 1;
        }

        float unitstep = fCUSTOM_VOLUME_MAPPING_STEP/dMaxLevels;
        if (vol < fCUSTOM_VOLUME_MAPPING_STEP/streamDescmIndexMax) {
            volume = array[0];
            vol = volume;
            return volume;
        }
        int Index = mapVol(vol, unitstep);
        float Remind = (1.0 - (float)vol/unitstep);
        if (Index != 0) {
            volume = ((array[Index]  - (array[Index] - array[Index-1]) * Remind)+0.5);
        } else {
            volume = 0;
        }
        MTK_ALOGVV("%s vol [%f] unitstep [%f] Index [%d] Remind [%f] volume [%f]", __FUNCTION__, vol, unitstep, Index, Remind, volume);
    }
    // -----clamp for volume
    if ( volume > 253.0) {
        volume = fCUSTOM_VOLUME_MAPPING_STEP;
    } else if ( volume <= array[0]) {
        volume = array[0];
    }
    vol = volume;
    MTK_ALOGVV("%s volume [%f] vol [%f]", __FUNCTION__, volume, vol);
    return volume;
}

// this function will map vol 0~100 , base on customvolume map to 0~255 , and do linear calculation to set mastervolume
float AudioPolicyManager::mapVoiceVoltoCustomVol(unsigned char array[], int volmin __unused, int volmax __unused, float &vol, int vol_stream_type)
{
    vol = (int)vol;
    float volume = 0.0;
//  StreamDescriptor &streamDesc = mStreams.valueFor((audio_stream_type_t)AUDIO_STREAM_VOICE_CALL);//mStreams[AUDIO_STREAM_VOICE_CALL];
    if (vol == 0) {
        volume = array[0];
    } else {
        int dMaxIndex = getStreamMaxLevels(AUDIO_STREAM_VOICE_CALL)-1;
        if (dMaxIndex < 0) {
            ALOGE("%s %d Error : dMaxIndex = %d", __FUNCTION__, __LINE__, dMaxIndex);
            dMaxIndex = 1;
        }
        if (vol >= fCUSTOM_VOLUME_MAPPING_STEP) {
            volume = array[dMaxIndex];
            MTK_ALOGVV("%s volumecheck stream = %d index = %d volume = %f", __FUNCTION__, AUDIO_STREAM_VOICE_CALL, dMaxIndex, volume);
        } else {
            double unitstep = fCUSTOM_VOLUME_MAPPING_STEP /dMaxIndex;
            int Index = mapping_Voice_vol(vol, unitstep);
            // boundary for array
            if (Index >= dMaxIndex) {
                Index = dMaxIndex;
            }
            float Remind = (1.0 - (float)vol/unitstep) ;
            if (Index != 0) {
                volume = (array[Index]  - (array[Index] - array[Index- 1]) * Remind)+0.5;
            } else {
                volume =0;
            }
            MTK_ALOGVV("%s volumecheck stream = %d index = %d volume = %f", __FUNCTION__, AUDIO_STREAM_VOICE_CALL, Index, volume);
            MTK_ALOGVV("%s dMaxIndex [%d] vol [%f] unitstep [%f] Index [%d] Remind [%f] volume [%f]", __FUNCTION__, dMaxIndex, vol, unitstep, Index, Remind, volume);
        }
    }

     if ( volume > CUSTOM_VOICE_VOLUME_MAX && vol_stream_type == CUSTOM_VOL_TYPE_VOICE_CALL) {
         volume = CUSTOM_VOICE_VOLUME_MAX;
     }
     else if ( volume > 253.0) {
        volume = fCUSTOM_VOLUME_MAPPING_STEP;
     }
     else if ( volume <= array[0]) {
         volume = array[0];
     }

     vol = volume;
     if ( vol_stream_type == CUSTOM_VOL_TYPE_VOICE_CALL) {
         float degradeDb = (CUSTOM_VOICE_VOLUME_MAX-vol)/CUSTOM_VOICE_ONEDB_STEP;
         MTK_ALOGVV("%s volume [%f] degradeDb [%f]", __FUNCTION__, volume, degradeDb);
         vol = fCUSTOM_VOLUME_MAPPING_STEP - (degradeDb * 4);
     }
     MTK_ALOGVV("%s volume [%f] vol [%f]", __FUNCTION__, volume, vol);
     return volume;
}

float AudioPolicyManager::computeCustomVolume(int stream, int index, audio_devices_t device)
{
    // check if force use exist , get output device for certain mode
    device_category deviceCategory = Volume::getDeviceCategory(device);
    // compute custom volume
    float volume =0.0;
    int volmax=0 , volmin =0,volumeindex =0;
    int custom_vol_device_mode,audiovolume_steamtype;
    int dMaxStepIndex = 0;

    MTK_ALOGVV("%s volumecheck stream = %d index = %d device = %d", __FUNCTION__, stream, index, device);

    if (mAudioPolicyVendorControl.getVoiceReplaceDTMFStatus() && stream == AUDIO_STREAM_DTMF) {
        // normalize new index from 0~15(audio) to 0~6(voice)
        int tempindex = index;
        float DTMFvolInt = (fCUSTOM_VOLUME_MAPPING_STEP * (index - mVolumeCurves->getVolumeIndexMin(AUDIO_STREAM_DTMF))) / (mVolumeCurves->getVolumeIndexMax(AUDIO_STREAM_DTMF) - mVolumeCurves->getVolumeIndexMin(AUDIO_STREAM_DTMF));
        index = (DTMFvolInt*(mVolumeCurves->getVolumeIndexMax(AUDIO_STREAM_VOICE_CALL) - mVolumeCurves->getVolumeIndexMin(AUDIO_STREAM_VOICE_CALL))/ (fCUSTOM_VOLUME_MAPPING_STEP)) + mVolumeCurves->getVolumeIndexMin(AUDIO_STREAM_VOICE_CALL);
        MTK_ALOGVV("volumecheck refine DTMF index [%d] to Voice index [%d]", tempindex, index);
        stream = (int) AUDIO_STREAM_VOICE_CALL;
    }

    if( isInCall()==true && mTty_Ctm != AUD_TTY_OFF ) {
        deviceCategory = Volume::getDeviceCategory(getNewDeviceForTty(device, mTty_Ctm));
        stream = (int) AUDIO_STREAM_VOICE_CALL;
    }

    float volInt = (fCUSTOM_VOLUME_MAPPING_STEP * (index - mVolumeCurves->getVolumeIndexMin((audio_stream_type_t)stream))) / (mVolumeCurves->getVolumeIndexMax((audio_stream_type_t)stream) - mVolumeCurves->getVolumeIndexMin((audio_stream_type_t)stream));

    if (deviceCategory == DEVICE_CATEGORY_SPEAKER) {
        custom_vol_device_mode = CUSTOM_VOLUME_SPEAKER_MODE;
        if ((device & AUDIO_DEVICE_OUT_WIRED_HEADSET)||
             (device & AUDIO_DEVICE_OUT_WIRED_HEADPHONE))
                custom_vol_device_mode = CUSTOM_VOLUME_HEADSET_SPEAKER_MODE;
    } else if (deviceCategory == DEVICE_CATEGORY_HEADSET) {
        custom_vol_device_mode = CUSTOM_VOLUME_HEADSET_MODE;
    } else if (deviceCategory == DEVICE_CATEGORY_EARPIECE) {
        custom_vol_device_mode = CUSTOM_VOLUME_NORMAL_MODE;
    } else {
        custom_vol_device_mode = CUSTOM_VOLUME_HEADSET_SPEAKER_MODE;
    }

    if ((stream == (int) AUDIO_STREAM_VOICE_CALL) && (mEngine->getPhoneState() == AUDIO_MODE_IN_COMMUNICATION)) {
        audiovolume_steamtype = (int) CUSTOM_VOL_TYPE_SIP;
    } else if (stream >= (int) AUDIO_STREAM_VOICE_CALL && stream < (int) AUDIO_STREAM_CNT) {
        audiovolume_steamtype = stream;
    } else {
        audiovolume_steamtype = (int) CUSTOM_VOL_TYPE_MUSIC;
        ALOGE("%s %d Error : audiovolume_steamtype = %d",__FUNCTION__,__LINE__,audiovolume_steamtype);
    }

    dMaxStepIndex = getStreamMaxLevels(audiovolume_steamtype) - 1;

    if (dMaxStepIndex > CUSTOM_AUDIO_MAX_VOLUME_STEP - 1) {
        ALOGE("%s %d Error : dMaxStepIndex = %d", __FUNCTION__, __LINE__, dMaxStepIndex);
        dMaxStepIndex = CUSTOM_AUDIO_MAX_VOLUME_STEP - 1;
    } else if (dMaxStepIndex < 0) {
        ALOGE("%s %d Error : dMaxStepIndex = %d", __FUNCTION__, __LINE__, dMaxStepIndex);
        dMaxStepIndex = 0;
    }

    volmax =mAudioCustVolumeTable.audiovolume_steamtype[audiovolume_steamtype][custom_vol_device_mode][dMaxStepIndex];
    volmin = mAudioCustVolumeTable.audiovolume_steamtype[audiovolume_steamtype][custom_vol_device_mode][0];
    MTK_ALOGVV("%s audiovolume_steamtype %d custom_vol_device_mode %d stream %d", __FUNCTION__, audiovolume_steamtype, custom_vol_device_mode, audiovolume_steamtype);
    MTK_ALOGVV("%s getStreamMaxLevels(stream) %d volmax %d volmin %d volInt %f index %d", __FUNCTION__, getStreamMaxLevels(audiovolume_steamtype), volmax, volmin, volInt, index);
    volume = mapVoltoCustomVol(mAudioCustVolumeTable.audiovolume_steamtype[audiovolume_steamtype][custom_vol_device_mode], volmin, volmax,volInt, audiovolume_steamtype);

    volume = linearToLog(volInt);
    MTK_ALOGV("stream = %d after computeCustomVolume , volInt = %f volume = %f volmin = %d volmax = %d", audiovolume_steamtype, volInt, volume, volmin, volmax);
    return volume;
}

void AudioPolicyManager::LoadCustomVolume()
{
    //ALOGD("LoadCustomVolume Audio_Ver1_Custom_Volume");
#ifdef MTK_AUDIO_GAIN_TABLE
#ifndef MTK_NEW_VOL_CONTROL
    ALOGD("%s(), load GainTable", __FUNCTION__);
    android::GetAudioGainTableParamFromNV(&mCustomGainTableVolume);
#endif
#else
    //android::GetVolumeVer1ParamFromNV (&Audio_Ver1_Custom_Volume);
    mAudioCustVolumeTable.bRev = CUSTOM_VOLUME_REV_1;
    mAudioCustVolumeTable.bReady = 0;

    MTK_ALOGVV("B4 Update");
    for (int i=0;i<CUSTOM_NUM_OF_VOL_TYPE;i++) {
        MTK_ALOGVV("StreamType %d", i);
        for (int j=0;j<CUSTOM_NUM_OF_VOL_MODE;j++) {
            MTK_ALOGVV("DeviceType %d", j);
            for (int k=0;k<CUSTOM_AUDIO_MAX_VOLUME_STEP;k++) {
                MTK_ALOGVV("[IDX]:[Value] %d, %d", k, mAudioCustVolumeTable.audiovolume_steamtype[i][j][k]);
            }
        }
    }
    mpClientInterface->getCustomAudioVolume(&mAudioCustVolumeTable);
    if (mAudioCustVolumeTable.bReady!=0) {
        ALOGD("mUseCustomVolume true");
        mAudioPolicyVendorControl.setCustomVolumeStatus(true);
    } else {
        ALOGD("mUseCustomVolume false");
    }
    MTK_ALOGVV("After Update");
    for (int i=0;i<CUSTOM_NUM_OF_VOL_TYPE;i++) {
        MTK_ALOGVV("StreamType %d", i);
        for (int j=0;j<CUSTOM_NUM_OF_VOL_MODE;j++) {
            MTK_ALOGVV("DeviceType %d", j);
            for (int k=0;k<CUSTOM_AUDIO_MAX_VOLUME_STEP;k++) {
                MTK_ALOGVV("[IDX]:[Value] %d, %d", k, mAudioCustVolumeTable.audiovolume_steamtype[i][j][k]);
            }
        }
    }
#endif
}

bool AudioPolicyManager::isFMDirectMode(const sp<AudioPatch>& patch)
{
#if defined(MTK_AUDIO) && defined(SUPPORT_ANDROID_FM_PLAYER)

    if (patch->mPatch.sources[0].type == AUDIO_PORT_TYPE_DEVICE &&
        patch->mPatch.sinks[0].type == AUDIO_PORT_TYPE_DEVICE &&
        (patch->mPatch.sources[0].ext.device.type == AUDIO_DEVICE_IN_FM_TUNER)) {
        return true;
    } else {
        return false;
    }
#else
    return false;
#endif
}

bool AudioPolicyManager::isFMActive(void)
{
#if defined(MTK_AUDIO) && defined(SUPPORT_ANDROID_FM_PLAYER)

    for (ssize_t i = 0; i < (ssize_t)mAudioPatches.size(); i++) {
        ALOGV("%s size %zu/ %zu", __FUNCTION__, i, mAudioPatches.size());
        sp<AudioPatch> patchDesc = mAudioPatches.valueAt(i);
        if (isFMDirectMode(patchDesc)||
            (patchDesc->mPatch.sources[0].type == AUDIO_PORT_TYPE_DEVICE
            &&patchDesc->mPatch.sources[0].ext.device.type == AUDIO_DEVICE_IN_FM_TUNER)) {
            ALOGV("FM Active");
            return true;
        }
    }
#endif
    return false;
}

audio_io_handle_t AudioPolicyManager::getPrimaryFastOutput()
{
    for (size_t i = 0; i < mOutputs.size(); i++) {
        sp<SwAudioOutputDescriptor> outputDesc = mOutputs.valueAt(i);
        if (!outputDesc->isDuplicated() && outputDesc->mFlags & AUDIO_OUTPUT_FLAG_FAST) {
            return mOutputs.keyAt(i);
        }
    }
    return 0;
}

audio_io_handle_t AudioPolicyManager::getOffloadOutput(void)
{
    for (size_t i = 0; i < mOutputs.size(); i++) {
        sp<SwAudioOutputDescriptor> outputDesc = mOutputs.valueAt(i);
        if (!outputDesc->isDuplicated() && outputDesc->mFlags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) {
            return mOutputs.keyAt(i);
        }
    }
    return 0;
}

audio_devices_t AudioPolicyManager::getNewDeviceForTty(audio_devices_t OutputDevice, tty_mode_t tty_mode)
{
    audio_devices_t OutputDeviceForTty = AUDIO_DEVICE_NONE;

    if (OutputDevice & AUDIO_DEVICE_OUT_SPEAKER) {
        if (tty_mode == AUD_TTY_VCO) {
            ALOGD("%s(), speaker, TTY_VCO", __FUNCTION__);
#if defined(ENABLE_EXT_DAC) || defined(ALL_USING_VOICEBUFFER_INCALL)
            OutputDeviceForTty = AUDIO_DEVICE_OUT_EARPIECE;
#else
            OutputDeviceForTty = AUDIO_DEVICE_OUT_WIRED_HEADSET;
#endif
        } else if (tty_mode == AUD_TTY_HCO) {
            ALOGD("%s(), speaker, TTY_HCO", __FUNCTION__);
#if defined(ENABLE_EXT_DAC) || defined(ALL_USING_VOICEBUFFER_INCALL)
            OutputDeviceForTty = AUDIO_DEVICE_OUT_EARPIECE;
#else
            OutputDeviceForTty = AUDIO_DEVICE_OUT_SPEAKER;
#endif
        } else if (tty_mode == AUD_TTY_FULL) {
            ALOGD("%s(), speaker, TTY_FULL", __FUNCTION__);
#if defined(ENABLE_EXT_DAC) || defined(ALL_USING_VOICEBUFFER_INCALL)
            OutputDeviceForTty = AUDIO_DEVICE_OUT_EARPIECE;
#else
            OutputDeviceForTty = AUDIO_DEVICE_OUT_WIRED_HEADSET;
#endif
        }
    } else if ((OutputDevice == AUDIO_DEVICE_OUT_WIRED_HEADSET) ||
             (OutputDevice == AUDIO_DEVICE_OUT_WIRED_HEADPHONE)) {
        if (tty_mode == AUD_TTY_VCO) {
            ALOGD("%s(), headset, TTY_VCO", __FUNCTION__);
#if defined(ENABLE_EXT_DAC) || defined(ALL_USING_VOICEBUFFER_INCALL)
            OutputDeviceForTty = AUDIO_DEVICE_OUT_EARPIECE;
#else
            OutputDeviceForTty = AUDIO_DEVICE_OUT_WIRED_HEADSET;
#endif
        } else if (tty_mode == AUD_TTY_HCO) {
            ALOGD("%s(), headset, TTY_HCO", __FUNCTION__);
            OutputDeviceForTty = AUDIO_DEVICE_OUT_EARPIECE;
        } else if (tty_mode == AUD_TTY_FULL) {
            ALOGD("%s(), headset, TTY_FULL", __FUNCTION__);
#if defined(ENABLE_EXT_DAC) || defined(ALL_USING_VOICEBUFFER_INCALL)
            OutputDeviceForTty = AUDIO_DEVICE_OUT_EARPIECE;
#else
            OutputDeviceForTty = AUDIO_DEVICE_OUT_WIRED_HEADSET;
#endif
        }
    }

    ALOGD("getNewDeviceForTty() tty_mode=%d, OutputDevice=0x%x, OutputDeviceForTty=0x%x", tty_mode, OutputDevice, OutputDeviceForTty);
    return OutputDeviceForTty;
}

#else
float AudioPolicyManager::computeVolume(audio_stream_type_t stream __unused,
                                            int index __unused,
                                            audio_devices_t device __unused,
                                            const sp<AudioOutputDescriptor>& outputDesc __unused)
{
    return 0.0;
}

audio_io_handle_t AudioPolicyManager::getPrimaryFastOutput()
{
    return 0;
}

audio_io_handle_t AudioPolicyManager::getOffloadOutput(void)
{
    return 0;
}

bool AudioPolicyManager::isFMDirectMode(const sp<AudioPatch>& patch __unused)
{
    return false;
}

bool AudioPolicyManager::isFMActive(void)
{
    return false;
}


status_t AudioPolicyManager::SetPolicyManagerParameters(int par1 __unused,int par2 __unused,int par3 __unused,int par4 __unused)
{
    return INVALID_OPERATION;
}

float AudioPolicyManager::linearToLog(int volume __unused)
{
    return 0.0;
}
int AudioPolicyManager::logToLinear(float volume __unused)
{
    return 0;
}

int AudioPolicyManager::mapVol(float &vol __unused, float unitstep __unused)
{
    return 0;
}

float AudioPolicyManager::computeCustomVolume(int stream __unused, int index __unused, audio_devices_t device __unused)
{
    return 0.0;
}

int AudioPolicyManager::getStreamMaxLevels(int  stream __unused)
{
    return 0;
}

float AudioPolicyManager::mapVoiceVoltoCustomVol(unsigned char array[] __unused, int volmin __unused, int volmax __unused, float &vol __unused, int vol_stream_type __unused)
{
    return 0.0;
}

float AudioPolicyManager::mapVoltoCustomVol(unsigned char array[] __unused, int volmin __unused, int volmax __unused,float &vol __unused, int stream __unused)
{
    return 0.0;
}

void AudioPolicyManager::LoadCustomVolume(void)
{
    return;
}

float AudioPolicyManager::computeGainTableCustomVolume(int stream __unused, int index __unused, audio_devices_t device __unused)
{
    return 0.0;
}
int AudioPolicyManager::selectGainTableActiveStream(int requestStream __unused)
{
    return AUDIO_STREAM_DEFAULT;
}
status_t AudioPolicyManager::checkAndSetGainTableAnalogGain(int stream __unused, int index __unused, const sp<AudioOutputDescriptor>& outputDesc __unused,audio_devices_t device __unused,
                                                           int delayMs __unused, bool force __unused)
{
    return INVALID_OPERATION;
}

uint32_t AudioPolicyManager::GetSampleRateIndex(uint32_t sampleRate __unused)
{
    return 0;
}

uint32_t AudioPolicyManager::GetSampleRateCount()
{
    return 0;
}

bool AudioPolicyManager::CheckFirstActive(void)
{
    return false;
}

bool AudioPolicyManager::CheckStreamActive(void)
{
    return false;
}

void AudioPolicyManager::DumpSampleRateArray(void)
{

}

bool AudioPolicyManager::SetFMIndirectMode(uint32_t SampleRate __unused)
{
    return false;
}

bool AudioPolicyManager::ReleaseFMIndirectMode(uint32_t SampleRate __unused)
{
    return true;
}
uint32_t AudioPolicyManager::GetFirstTrackSampleRate(void)
{
    return 0;
}

bool AudioPolicyManager::PrimaryDeviceSupportSampleRate(uint32_t SampleRate __unused)
{
    return true;
}


bool AudioPolicyManager::PrimarySupportSampleRate(uint32_t SampleRate __unused)
{
    return false;
}

status_t AudioPolicyManager::PolicyFirstStart(audio_stream_type_t stream __unused, uint32_t sampleRate __unused)
{
    return NO_ERROR;
}

int AudioPolicyManager::PolicyPropertyCheck(void)
{
    return 0;
}


status_t AudioPolicyManager::PolicyRestore(void)
{
     return NO_ERROR;
}

status_t AudioPolicyManager::PolicyForceReplace(audio_stream_type_t stream __unused, uint32_t sampleRate __unused)
{
    return NO_ERROR;
}

status_t AudioPolicyManager::SampleRatePolicy(audio_stream_type_t stream __unused, uint32_t sampleRate __unused)
{
    return NO_ERROR;
}


status_t AudioPolicyManager::AddSampleRateArray(audio_stream_type_t stream __unused, uint32_t sampleRate __unused)
{
    return NO_ERROR;
}

status_t AudioPolicyManager::RemoveSampleRateArray(audio_stream_type_t stream __unused, uint32_t sampleRate __unused)
{
    return NO_ERROR;
}

status_t AudioPolicyManager::SampleRateRequestFocus(audio_io_handle_t output __unused,
                                                      audio_stream_type_t stream __unused,
                                                      int *samplerate __unused)
{
    return NO_ERROR;
}

status_t AudioPolicyManager::SampleRateUnrequestFocus(audio_io_handle_t output __unused,
                                                      audio_stream_type_t stream __unused,
                                                      int samplerate __unused)
{
    return NO_ERROR;
}

status_t AudioPolicyManager::StartOutputSamplerate(audio_io_handle_t output __unused,
                                                      audio_stream_type_t stream __unused,
                                                      audio_session_t session __unused, int samplerate __unused)
{
    return INVALID_OPERATION;
}

status_t AudioPolicyManager::StopOutputSamplerate(audio_io_handle_t output __unused,
                                                     audio_stream_type_t stream __unused,
                                                     audio_session_t session __unused, int samplerate __unused)
{
    return INVALID_OPERATION;
}

void AudioPolicyManager::InitSamplerateArray(uint32_t init_sample_rate __unused)
{

}

audio_devices_t AudioPolicyManager::getNewDeviceForTty(audio_devices_t OutputDevice __unused, tty_mode_t tty_mode __unused)
{
    return AUDIO_DEVICE_NONE;
}

#endif
}; // namespace android
