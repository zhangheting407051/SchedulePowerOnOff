package com.mediatek.internal.telephony;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.os.AsyncResult;
import android.os.Handler;
import android.os.Message;
import android.os.SystemProperties;
import android.os.UserHandle;
import android.telephony.Rlog;
import android.telephony.TelephonyManager;
import android.net.ConnectivityManager;

import com.android.internal.telephony.CommandsInterface.RadioState;
import com.android.internal.telephony.Phone;
import com.android.internal.telephony.PhoneFactory;
import com.android.internal.telephony.TelephonyIntents;
import com.mediatek.internal.telephony.worldphone.WorldMode;
import com.mediatek.internal.telephony.worldphone.WorldPhoneUtil;

import java.util.concurrent.atomic.AtomicBoolean;
/**
 * This class fix the bug turn on/off flightmode frenquently.
 */
public class AirplaneRequestHandler extends Handler {
    private static final String LOG_TAG = "AirplaneRequestHandler";
    private Context mContext;
    private Boolean mPendingAirplaneModeRequest;
    private int mPhoneCount;
    private boolean mNeedIgnoreMessageForChangeDone;
    private boolean mForceSwitch;
    private boolean mNeedIgnoreMessageForWait;

    // For set Modem Power
    private boolean mPowerModem;
    private ModemPowerMessage[] mModemPowerMessages;
    private boolean mIsRadioUnavailable = false;

    private static final int EVENT_GSM_RADIO_CHANGE_FOR_OFF = 100;
    private static final int EVENT_GSM_RADIO_CHANGE_FOR_AVALIABLE = 101;
    private static final int EVENT_WAIT_RADIO_CHANGE_FOR_AVALIABLE = 102;
    private static final int EVENT_WAIT_RADIO_CHANGE_UNAVALIABLE_TO_AVALIABLE = 103;
    private static final int EVENT_SET_MODEM_POWER_DONE = 104;

    private static AtomicBoolean sInSwitching = new AtomicBoolean(false);

    private boolean mHasRegisterWorldModeReceiver = false;
    private BroadcastReceiver mWorldModeReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();
            int wmState = WorldMode.MD_WM_CHANGED_UNKNOWN;
            //log("mWorldModeReceiver: action = " + action);
            if (TelephonyIntents.ACTION_WORLD_MODE_CHANGED.equals(action)) {
                wmState = intent.getIntExtra(TelephonyIntents.EXTRA_WORLD_MODE_CHANGE_STATE,
                        WorldMode.MD_WM_CHANGED_UNKNOWN);
                //log("mWorldModeReceiver: wmState = " + wmState);
                if (mHasRegisterWorldModeReceiver) {
                    if (wmState == WorldMode.MD_WM_CHANGED_END) {
                        unRegisterWorldModeReceiver();
                        sInSwitching.set(false);
                        checkPendingRequest();
                    }
                }
            }
        }
    };

    protected boolean allowSwitching() {
        if (sInSwitching.get()  && !mForceSwitch) {
            return false;
        }
        return true;
    }

    protected void pendingAirplaneModeRequest(boolean enabled) {
        log("pendingAirplaneModeRequest, enabled = " + enabled);
        mPendingAirplaneModeRequest = new Boolean(enabled);
    }

    /**
     * The Airplane mode change request handler.
     * @param context the context
     * @param phoneCount the phone count
     */
    public AirplaneRequestHandler(Context context, int phoneCount) {
        mContext = context;
        mPhoneCount = phoneCount;
    }

    protected void monitorRadioChangeDone(boolean power) {
        mNeedIgnoreMessageForChangeDone = false;
        //log("monitorRadioChangeDone, power = " + power
        //    + " mNeedIgnoreMessageForChangeDone = " + mNeedIgnoreMessageForChangeDone);
        sInSwitching.set(true);
        Phone phone;
        for (int i = 0; i < mPhoneCount; i++) {
            phone = PhoneFactory.getPhone(i);
            if (phone == null) {
                continue;
            }
            if (power) {
                phone.mCi.registerForRadioStateChanged(this,
                        EVENT_GSM_RADIO_CHANGE_FOR_AVALIABLE , null);
            } else {
                phone.mCi.registerForRadioStateChanged(this,
                        EVENT_GSM_RADIO_CHANGE_FOR_OFF, null);
            }
        }
    }

    @Override
    public void handleMessage(Message msg) {
        ///M: Add for wifi only device. @{
        boolean isWifiOnly = isWifiOnly();
        ///  @}
        Phone phone;
        switch (msg.what) {
        case EVENT_GSM_RADIO_CHANGE_FOR_OFF:
            if (!mNeedIgnoreMessageForChangeDone) {
                if (msg.what == EVENT_GSM_RADIO_CHANGE_FOR_OFF) {
                    log("handle EVENT_GSM_RADIO_CHANGE_FOR_OFF");
                }
                for (int i = 0; i < mPhoneCount; i++) {
                    phone = PhoneFactory.getPhone(i);
                    if (phone == null) {
                        return;
                    }
                    ///M: Add for wifi only device, don't judge radio off. @{
                    if (isWifiOnly) {
                        log("wifi-only, don't judge radio off");
                        break;
                    }
                    ///  @}
                    if (!isRadioOff(i)) {
                        //log("radio state change, radio not off, phoneId = " + i);
                        return;
                    }
                }
                //log("All radio off");
                sInSwitching.set(false);
                unMonitorAirplaneChangeDone(true);
                checkPendingRequest();
            }
            break;
        case EVENT_GSM_RADIO_CHANGE_FOR_AVALIABLE:
            if (!mNeedIgnoreMessageForChangeDone) {
                if (msg.what == EVENT_GSM_RADIO_CHANGE_FOR_AVALIABLE) {
                    log("handle EVENT_GSM_RADIO_CHANGE_FOR_AVALIABLE");
                }
                for (int i = 0; i < mPhoneCount; i++) {
                    phone = PhoneFactory.getPhone(i);
                    if (phone == null) {
                        return;
                    }
                    ///M: Add for wifi only device, don't judge radio avaliable. @{
                    if (isWifiOnly) {
                        log("wifi-only, don't judge radio avaliable");
                        break;
                    }
                    ///  @}
                    if (!isRadioAvaliable(i)) {
                        //log("radio state change, radio not avaliable, phoneId = " + i);
                        return;
                    }
                }
                //log("All radio avaliable");
                sInSwitching.set(false);
                unMonitorAirplaneChangeDone(false);
                checkPendingRequest();
            }
            break;
        case EVENT_WAIT_RADIO_CHANGE_FOR_AVALIABLE:
            if (!mNeedIgnoreMessageForWait) {
                log("handle EVENT_WAIT_RADIO_CHANGE_FOR_AVALIABLE");
                if (!isRadioAvaliable()) {
                    return;
                }
                //log("All radio avaliable");
                unWaitRadioAvaliable();
                sInSwitching.set(false);
                checkPendingRequest();
            }
            break;
        case EVENT_WAIT_RADIO_CHANGE_UNAVALIABLE_TO_AVALIABLE:
            log("handle EVENT_WAIT_RADIO_CHANGE_UNAVALIABLE_TO_AVALIABLE");
            if (!mNeedIgnoreMessageForChangeDone && mPowerModem) {
                if (isWifiOnly) {
                    log("wifi-only, don't judge radio avaliable");
                } else if (!isRadioOn()) {
                    if (!mIsRadioUnavailable) {
                        if (isRadioUnavailable()) {
                            log("All radio unavaliable");
                            mIsRadioUnavailable = true;
                        }
                        return;
                    }

                    if (!isRadioAvaliable()) {
                        return;
                    }
                    //log("All radio avaliable");
                }

                sInSwitching.set(false);
                unMonitorAirplaneChangeDone(false);
                checkPendingRequest();
            }
            break;
        case EVENT_SET_MODEM_POWER_DONE:
            log("[SMP]handle EVENT_SET_MODEM_POWER_DONE -> " + (mPowerModem ? "ON" : "OFF"));
            if (!mPowerModem) {
                final AsyncResult ar = (AsyncResult) msg.obj;
                final ModemPowerMessage message = (ModemPowerMessage) ar.userObj;
                log("[SMP]handleModemPowerMessage, message:" + message);
                if (ar.exception == null) {
                    if (ar.result != null) {
                        log("[SMP]handleModemPowerMessage, result:" + ar.result);
                    }
                } else {
                    // error handle
                    log("[SMP]handleModemPowerMessage, Unhandle ar.exception:" + ar.exception);
                }

                message.isFinish = true;

                if (!isSetModemPowerFinish()) {
                    return;
                }

                //log("[SMP]handleModemPowerMessage, isSetModemPowerFinish");
                cleanModemPowerMessage();

                sInSwitching.set(false);
                unMonitorAirplaneChangeDone(true);
                checkPendingRequest();
            }
            break;
          default:
            break;
        }
    }

    private boolean isRadioAvaliable(int phoneId) {
        Phone phone;
        phone = PhoneFactory.getPhone(phoneId);
        if (phone == null) {
            return false;
        }

        log("phoneId = " + phoneId + ", RadioState="
                + phone.mCi.getRadioState());
        return phone.mCi.getRadioState() != RadioState.RADIO_UNAVAILABLE;
    }

    private boolean isRadioOff(int phoneId) {
        Phone phone;
        phone = PhoneFactory.getPhone(phoneId);
        if (phone == null) {
            return false;
        }

        log("phoneId = " + phoneId + ", RadioState="
                + phone.mCi.getRadioState());
        return phone.mCi.getRadioState() == RadioState.RADIO_OFF;
    }

    private boolean isRadioOn(int phoneId) {
        Phone phone;
        phone = PhoneFactory.getPhone(phoneId);
        if (phone == null) {
            return false;
        }

        return phone.mCi.getRadioState() == RadioState.RADIO_ON;
    }

    private void checkPendingRequest() {
        log("checkPendingRequest, mPendingAirplaneModeRequest = " + mPendingAirplaneModeRequest);
        if (mPendingAirplaneModeRequest != null) {
            Boolean pendingAirplaneModeRequest = mPendingAirplaneModeRequest;
            mPendingAirplaneModeRequest = null;
            RadioManager.getInstance().notifyAirplaneModeChange(
                    pendingAirplaneModeRequest.booleanValue());
        }
    }

    protected void unMonitorAirplaneChangeDone(boolean airplaneMode) {
        mNeedIgnoreMessageForChangeDone = true;
        Intent intent = new Intent(RadioManager.ACTION_AIRPLANE_CHANGE_DONE);
        intent.putExtra(RadioManager.EXTRA_AIRPLANE_MODE, airplaneMode);
        mContext.sendBroadcastAsUser(intent, UserHandle.ALL);

        Phone phone;
        for (int i = 0; i < mPhoneCount; i++) {
            phone = PhoneFactory.getPhone(i);
            if (phone == null) {
                continue;
            }
            phone.mCi.unregisterForRadioStateChanged(this);
            log("unMonitorAirplaneChangeDone, for gsm phone,  phoneId = " + i);
        }
    }

    /**
     * Set Whether force allow airplane mode change.
     * @return true or false
     */
    public void setForceSwitch(boolean forceSwitch) {
        mForceSwitch = forceSwitch;
        log("setForceSwitch, forceSwitch =" + forceSwitch);
    }

    protected boolean waitForReady(boolean enabled) {
        if (waitRadioAvaliable(enabled)) {
            //log("waitForReady, wait radio avaliable");
            return true;
        } else if (waitWorlModeSwitching(enabled)) {
            //log("waitForReady, wait world mode switching");
            return true;
        }
        else {
            return false;
        }
    }

    private boolean waitRadioAvaliable(boolean enabled) {
        final boolean wait = isCdmaLteDcSupport() && !isWifiOnly() && !isRadioAvaliable();
        log("waitRadioAvaliable, enabled=" + enabled + ", wait=" + wait);
        if (wait) {
            // pending
            pendingAirplaneModeRequest(enabled);

            // wait for radio avaliable
            mNeedIgnoreMessageForWait = false;
            sInSwitching.set(true);
            Phone phone;
            for (int i = 0; i < mPhoneCount; i++) {
                phone = PhoneFactory.getPhone(i);
                if (phone == null) {
                    continue;
                }
                phone.mCi.registerForRadioStateChanged(this,
                        EVENT_WAIT_RADIO_CHANGE_FOR_AVALIABLE, null);
            }
        }
        return wait;
    }

    private void unWaitRadioAvaliable() {
        mNeedIgnoreMessageForWait = true;
        Phone phone;
        for (int i = 0; i < mPhoneCount; i++) {
            phone = PhoneFactory.getPhone(i);
            if (phone == null) {
                continue;
            }
            phone.mCi.unregisterForRadioStateChanged(this);
            log("unWaitRadioAvaliable, for gsm phone,  phoneId = " + i);
        }
    }

    private boolean isRadioOn() {
        boolean isRadioOn = true;
        for (int i = 0; i < mPhoneCount; i++) {
            if (!isRadioOn(i)) {
                isRadioOn = false;
                break;
            }
        }
        return isRadioOn;
    }

    private boolean isRadioAvaliable() {
        boolean isRadioAvaliable = true;
        for (int i = 0; i < mPhoneCount; i++) {
            if (!isRadioAvaliable(i)) {
                log("isRadioAvaliable=false, phoneId = " + i);
                isRadioAvaliable = false;
                break;
            }
        }
        return isRadioAvaliable;
    }

    private boolean isRadioUnavailable() {
        boolean isRadioUnavailable = true;
        for (int i = 0; i < mPhoneCount; i++) {
            if (isRadioAvaliable(i)) {
                log("isRadioUnavailable=false, phoneId = " + i);
                isRadioUnavailable = false;
                break;
            }
        }
        return isRadioUnavailable;
    }

    private boolean waitWorlModeSwitching(boolean enabled) {
        final boolean wait = isCdmaLteDcSupport() && !isWifiOnly()
                && WorldPhoneUtil.isWorldPhoneSwitching();
        log("waitWorlModeSwitching, enabled=" + enabled + ", wait=" + wait);
        if (wait) {
            // pending
            pendingAirplaneModeRequest(enabled);

            // wait for world mode switching
            sInSwitching.set(true);

            if (!mHasRegisterWorldModeReceiver) {
                registerWorldModeReceiver();
            }
        }
        return wait;
    }

    private void registerWorldModeReceiver() {
        final IntentFilter filter = new IntentFilter();
        filter.addAction(TelephonyIntents.ACTION_WORLD_MODE_CHANGED);
        mContext.registerReceiver(mWorldModeReceiver, filter);
        mHasRegisterWorldModeReceiver = true;
    }

    private void unRegisterWorldModeReceiver() {
        mContext.unregisterReceiver(mWorldModeReceiver);
        mHasRegisterWorldModeReceiver = false;
    }

    private boolean isWifiOnly() {
        final ConnectivityManager cm = (ConnectivityManager) mContext.getSystemService(
                Context.CONNECTIVITY_SERVICE);
        final boolean isWifiOnly = !cm.isNetworkSupported(ConnectivityManager.TYPE_MOBILE);
        return isWifiOnly;
    }

    private static final boolean isCdmaLteDcSupport() {
        if (SystemProperties.get("ro.boot.opt_c2k_lte_mode").equals("1") ||
                SystemProperties.get("ro.boot.opt_c2k_lte_mode").equals("2")) {
            return true;
        } else {
            return false;
        }
    }

    private static void log(String s) {
        Rlog.d(LOG_TAG, "[RadioManager] " + s);
    }

    protected final Message[] monitorModemPowerChangeDone(boolean power, int phoneBitMap,
            int mainCapabilityPhoneId) {
        mPowerModem = power;

        log("[SMP]monitorModemPowerChangeDone, Power:" + power + ", PhoneBitMap:" + phoneBitMap
                + ", mainCapabilityPhoneId:" + mainCapabilityPhoneId
                + ", mPhoneCount:" + mPhoneCount);

        mNeedIgnoreMessageForChangeDone = false;
        mIsRadioUnavailable = false;
        sInSwitching.set(true);

        final Message[] msgs = new Message[mPhoneCount];

        if (mPowerModem) {
            // [Modem Power On] wait radio change from unavaliable to avaliable
            Phone phone;
            for (int i = 0; i < mPhoneCount; i++) {
                phone = PhoneFactory.getPhone(i);
                if (phone == null) {
                    continue;
                }
                phone.mCi.registerForRadioStateChanged(this,
                        EVENT_WAIT_RADIO_CHANGE_UNAVALIABLE_TO_AVALIABLE, null);
            }
        } else {
            // [Modem Power Off] wait response message
            // Create Message
            final ModemPowerMessage[] messages = createMessage(power, phoneBitMap,
                    mainCapabilityPhoneId, mPhoneCount);
            mModemPowerMessages = messages;

            // Obtain Message
            for (int i = 0; i < messages.length; i++) {
                if (messages[i] != null) {
                    msgs[i] = obtainMessage(EVENT_SET_MODEM_POWER_DONE, messages[i]);
                }
            }
        }

        return msgs;
    }

    private final boolean isSetModemPowerFinish() {
        if (mModemPowerMessages != null) {
            for (int i = 0; i < mModemPowerMessages.length; i++) {
                if (mModemPowerMessages[i] != null) {
                    log("[SMP]isSetModemPowerFinish [" + i + "]: " + mModemPowerMessages[i]);
                    if (!mModemPowerMessages[i].isFinish) {
                        return false;
                    }
                } else {
                    log("[SMP]isSetModemPowerFinish [" + i + "]: MPMsg is null");
                }
            }
        }
        return true;
    }

    private final void cleanModemPowerMessage() {
        log("[SMP]cleanModemPowerMessage");
        if (mModemPowerMessages != null) {
            for (int i = 0; i < mModemPowerMessages.length; i++) {
                mModemPowerMessages[i] = null;
            }
            mModemPowerMessages = null;
        }
    }

    private static final ModemPowerMessage[] createMessage(
            boolean power, int phoneBitMap, int mainCapabilityPhoneId, int phoneCount) {
        final TelephonyManager.MultiSimVariants config =
                TelephonyManager.getDefault().getMultiSimConfiguration();
        log("[SMP]createMessage, config:" + config);

        final ModemPowerMessage[] msgs = new ModemPowerMessage[phoneCount];
        int phoneId = 0;
        switch (config) {
            case DSDS:
            case TSTS:
                phoneId = mainCapabilityPhoneId;
                msgs[phoneId] = new ModemPowerMessage(phoneId);
                break;
            case DSDA:
                for (int i = 0; i < phoneCount; i++) {
                    phoneId = i;
                    if ((phoneBitMap & (RadioManager.MODE_PHONE1_ONLY << i)) != 0) {
                        log("[SMP]createMessage, Power:" + power + ", phoneId:" + phoneId);
                        msgs[phoneId] = new ModemPowerMessage(phoneId);
                    }
                }
                break;
            default:
                phoneId = PhoneFactory.getDefaultPhone().getPhoneId();
                msgs[phoneId] = new ModemPowerMessage(phoneId);
                break;
        }

        for (int i = 0; i < phoneCount; i++) {
            if (msgs[i] != null) {
                log("[SMP]createMessage, [" + i + "]: " + msgs[i].toString());
            }
        }
        return msgs;
    }

    private static final class ModemPowerMessage {
        private final int mPhoneId;
        public boolean isFinish = false;

        public ModemPowerMessage(int phoneId) {
            this.mPhoneId = phoneId;
        }

        @Override
        public String toString() {
            return "MPMsg [mPhoneId=" + mPhoneId + ", isFinish=" + isFinish + "]";
        }
    }
}
