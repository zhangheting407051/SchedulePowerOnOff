/* Copyright Statement:
 *
 * This software/firmware and related documentation ("MediaTek Software") are
 * protected under relevant copyright laws. The information contained herein is
 * confidential and proprietary to MediaTek Inc. and/or its licensors. Without
 * the prior written permission of MediaTek inc. and/or its licensors, any
 * reproduction, modification, use or disclosure of MediaTek Software, and
 * information contained herein, in whole or in part, shall be strictly
 * prohibited.
 *
 * MediaTek Inc. (C) 2014. All rights reserved.
 *
 * BY OPENING THIS FILE, RECEIVER HEREBY UNEQUIVOCALLY ACKNOWLEDGES AND AGREES
 * THAT THE SOFTWARE/FIRMWARE AND ITS DOCUMENTATIONS ("MEDIATEK SOFTWARE")
 * RECEIVED FROM MEDIATEK AND/OR ITS REPRESENTATIVES ARE PROVIDED TO RECEIVER
 * ON AN "AS-IS" BASIS ONLY. MEDIATEK EXPRESSLY DISCLAIMS ANY AND ALL
 * WARRANTIES, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED
 * WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE OR
 * NONINFRINGEMENT. NEITHER DOES MEDIATEK PROVIDE ANY WARRANTY WHATSOEVER WITH
 * RESPECT TO THE SOFTWARE OF ANY THIRD PARTY WHICH MAY BE USED BY,
 * INCORPORATED IN, OR SUPPLIED WITH THE MEDIATEK SOFTWARE, AND RECEIVER AGREES
 * TO LOOK ONLY TO SUCH THIRD PARTY FOR ANY WARRANTY CLAIM RELATING THERETO.
 * RECEIVER EXPRESSLY ACKNOWLEDGES THAT IT IS RECEIVER'S SOLE RESPONSIBILITY TO
 * OBTAIN FROM ANY THIRD PARTY ALL PROPER LICENSES CONTAINED IN MEDIATEK
 * SOFTWARE. MEDIATEK SHALL ALSO NOT BE RESPONSIBLE FOR ANY MEDIATEK SOFTWARE
 * RELEASES MADE TO RECEIVER'S SPECIFICATION OR TO CONFORM TO A PARTICULAR
 * STANDARD OR OPEN FORUM. RECEIVER'S SOLE AND EXCLUSIVE REMEDY AND MEDIATEK'S
 * ENTIRE AND CUMULATIVE LIABILITY WITH RESPECT TO THE MEDIATEK SOFTWARE
 * RELEASED HEREUNDER WILL BE, AT MEDIATEK'S OPTION, TO REVISE OR REPLACE THE
 * MEDIATEK SOFTWARE AT ISSUE, OR REFUND ANY SOFTWARE LICENSE FEES OR SERVICE
 * CHARGE PAID BY RECEIVER TO MEDIATEK FOR SUCH MEDIATEK SOFTWARE AT ISSUE.
 *
 * The following software/firmware and/or related documentation ("MediaTek
 * Software") have been modified by MediaTek Inc. All revisions are subject to
 * any receiver's applicable license agreements with MediaTek Inc.
 */

package com.mediatek.internal.telephony;

import android.os.AsyncResult;
import android.os.Handler;
import android.os.Message;
import android.os.SystemProperties;
import android.os.UserHandle;
import android.provider.Settings;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.content.SharedPreferences;
import android.telephony.Rlog;
import android.telephony.SubscriptionManager;
import android.telephony.TelephonyManager;
import android.net.ConnectivityManager;
import android.net.wifi.WifiManager;

import com.android.ims.ImsConfig;
import com.android.ims.ImsManager;
import com.android.internal.telephony.CommandsInterface;
import com.android.internal.telephony.IccCardConstants;
import com.android.internal.telephony.Phone;
import com.android.internal.telephony.TelephonyIntents;
import com.android.internal.telephony.PhoneConstants;
import com.android.internal.telephony.PhoneFactory;
import com.android.internal.telephony.uicc.UiccController;

import java.util.concurrent.ConcurrentHashMap;
import java.util.Map.Entry;

import com.mediatek.internal.telephony.IRadioPower;

public class RadioManager extends Handler  {

    static final String LOG_TAG = "RadioManager";
    private static final String PREF_CATEGORY_RADIO_STATUS = "RADIO_STATUS";
    private static RadioManager sRadioManager;

    protected static final int MODE_PHONE1_ONLY = 1;
    private static final int MODE_PHONE2_ONLY = 2;
    private static final int MODE_PHONE3_ONLY = 4;
    private static final int MODE_PHONE4_ONLY = 8;

    protected static final int INVALID_PHONE_ID = -1;

    private static final int SIM_NOT_INITIALIZED = -1;
    protected static final int NO_SIM_INSERTED = 0;
    protected static final int SIM_INSERTED = 1;

    private static final boolean ICC_READ_NOT_READY = false;
    private static final boolean ICC_READ_READY = true;

    protected static final int INITIAL_RETRY_INTERVAL_MSEC = 200;

    protected static final boolean RADIO_POWER_OFF = false;
    protected static final boolean RADIO_POWER_ON = true;

    protected static final boolean MODEM_POWER_OFF = false;
    protected static final boolean MODEM_POWER_ON = true;

    protected static final boolean AIRPLANE_MODE_OFF = false;
    protected static final boolean AIRPLANE_MODE_ON = true;
    protected boolean mAirplaneMode = AIRPLANE_MODE_OFF;

    private static final int WIFI_ONLY_INIT = -1;
    private static final boolean WIFI_ONLY_MODE_OFF = false;
    private static final boolean WIFI_ONLY_MODE_ON = true;
    private boolean mWifiOnlyMode = WIFI_ONLY_MODE_OFF;
    private static final String ACTION_WIFI_ONLY_MODE_CHANGED =
        "android.intent.action.ACTION_WIFI_ONLY_MODE";

    protected static final String STRING_NO_SIM_INSERTED = "N/A";

    protected static final String PROPERTY_SILENT_REBOOT_MD1 = "gsm.ril.eboot";
    protected static final String PROPERTY_SILENT_REBOOT_MD2 = "gsm.ril.eboot.2";
    private static final String PROPERTY_SILENT_REBOOT_CDMA = "cdma.ril.eboot";
    private static final String MTK_C2K_SUPPORT = "ro.boot.opt_c2k_support";

    private static final String IS_NOT_SILENT_REBOOT = "0";
    protected static final String IS_SILENT_REBOOT = "1";
    private static final String REGISTRANTS_WITH_NO_NAME = "NO_NAME";

    // proprietary intent
    public static final String ACTION_FORCE_SET_RADIO_POWER =
        "com.mediatek.internal.telephony.RadioManager.intent.action.FORCE_SET_RADIO_POWER";
    private static final String WIFI_OFFLOAD_SERVICE_ON =
        "mediatek.intent.action.WFC_POWER_ON_MODEM";
    private static final String WIFI_OFFLOAD_SERVICE_ON_EXTRA = "mediatek:POWER_ON_MODEM";

    /// Handle WFC and Airplane mode special case @{
    protected static final String ACTION_AIRPLANE_CHANGE_DONE
                                    = "com.mediatek.intent.action.AIRPLANE_CHANGE_DONE";
    protected static final String EXTRA_AIRPLANE_MODE = "airplaneMode";

    protected static final int TO_SKIP_AND_FAKE_DONE = 0;
    protected static final int TO_SET_RADIO_POWER = 1;
    protected static final int TO_SET_MODEM_POWER = 2;
    /// @}

    protected int[] mSimInsertedStatus;
    private Context mContext;
    private int[] mInitializeWaitCounter;
    private CommandsInterface[] mCi;
    protected static SharedPreferences sIccidPreference;
    protected int mPhoneCount;
    protected int mBitmapForPhoneCount;
    //For checking if ECC call is on-going to bypass turning off radio
    protected boolean mIsEccCall;
    private int mSimModeSetting;
    private Runnable mNotifyMSimModeChangeRunnable;

    private boolean mIsWifiOn = false;

    // Record ipo shutdown request for those phone which in radio not available state,
    // should send ipo shutdown request again when radio state is available
    private boolean bIsQueueIpoShutdown;

    private boolean bIsQueueIpoPreboot;

    // Is in IPO shutdown, needs to block all radio power on request
    // ex. ECC do not recognize IPO shutdown, so it will force set radio power on after its timeout
    // in this scenario, modem will power-on wrongly: airplane on -> ecc call -> ipo shutdown
    // -> ecc trigger airplane off -> ecc timeout -> force set radio power on
    private boolean bIsInIpoShutdown;

    private ImsSwitchController mImsSwitchController = null;

    static protected ConcurrentHashMap<IRadioPower, String> mNotifyRadioPowerChange
            = new ConcurrentHashMap<IRadioPower, String>();

    protected static String[] PROPERTY_ICCID_SIM = {
        "ril.iccid.sim1",
        "ril.iccid.sim2",
        "ril.iccid.sim3",
        "ril.iccid.sim4",
    };

    protected static String[] PROPERTY_RADIO_OFF = {
        "ril.ipo.radiooff",
        "ril.ipo.radiooff.2",
    };

    // MD power off flag @{
    protected static String PROPERTY_RIL_MD_OFF = "ril.mdpoweroff";
    // @}

    /** events id definition */
    private static final int EVENT_RADIO_AVAILABLE = 1;
    private static final int EVENT_VIRTUAL_SIM_ON = 2;

    public static RadioManager init(Context context, int phoneCount, CommandsInterface[] ci) {
        synchronized (RadioManager.class) {
            if (sRadioManager == null) {
                sRadioManager =  new RadioManager(context, phoneCount, ci);
            }
            return sRadioManager;
        }
    }

    private AirplaneRequestHandler mAirplaneRequestHandler;
    /**
     * @internal
     */
    public static RadioManager getInstance() {
        synchronized (RadioManager.class) {
            return sRadioManager;
        }
    }

    protected RadioManager(Context context , int phoneCount, CommandsInterface[] ci) {

        int airplaneMode = Settings.Global.getInt(context.getContentResolver(),
                Settings.Global.AIRPLANE_MODE_ON, 0);
        int wifionlyMode = ImsManager.getWfcMode(context);

        // WFC modem power control: reference platform WFC capability to sync mIsWifiOn. @{
        if (ImsManager.isWfcEnabledByPlatform(context)) {
            log("initial actual wifi state when wifi calling is on");
            WifiManager wiFiManager = (WifiManager) context.getSystemService(Context.WIFI_SERVICE);
            mIsWifiOn = (wiFiManager.isWifiEnabled() == true) ? true : false;
        }
        // @}

        log("Initialize RadioManager under airplane mode:" + airplaneMode +
            " wifi only mode:" + wifionlyMode + " wifi mode: " + mIsWifiOn);

        mSimInsertedStatus = new int[phoneCount];
        for (int i = 0; i < phoneCount; i++) {
            mSimInsertedStatus[i] = SIM_NOT_INITIALIZED;
        }
        mInitializeWaitCounter = new int[phoneCount];
        for (int i = 0; i < phoneCount; i++) {
            mInitializeWaitCounter[i] = 0;
        }
        mNotifyMSimModeChangeRunnable = new MSimModeChangeRunnable(3);

        mContext = context;
        mAirplaneMode = ((airplaneMode == 0) ? AIRPLANE_MODE_OFF : AIRPLANE_MODE_ON);

        mWifiOnlyMode = (ImsConfig.WfcModeFeatureValueConstants.WIFI_ONLY == wifionlyMode);

        mCi = ci;
        mPhoneCount = phoneCount;
        mBitmapForPhoneCount = convertPhoneCountIntoBitmap(phoneCount);
        sIccidPreference = mContext.getSharedPreferences(PREF_CATEGORY_RADIO_STATUS, 0);
        mSimModeSetting = Settings.Global.getInt(context.getContentResolver(),
                              Settings.Global.MSIM_MODE_SETTING, mBitmapForPhoneCount);
        mImsSwitchController = new ImsSwitchController(mContext, mPhoneCount, mCi);

        if (!SystemProperties.get("ro.mtk_bsp_package").equals("1")) {
            log("Not BSP Package, register intent!!!");
            IntentFilter filter = new IntentFilter();
            filter.addAction(TelephonyIntents.ACTION_SIM_STATE_CHANGED);
            filter.addAction(ACTION_FORCE_SET_RADIO_POWER);
            filter.addAction(ACTION_WIFI_ONLY_MODE_CHANGED);
            filter.addAction(WIFI_OFFLOAD_SERVICE_ON);
            //filter.addAction(WifiManager.WIFI_STATE_CHANGED_ACTION);
            mContext.registerReceiver(mIntentReceiver, filter);

            // For virtual SIM
            for (int i = 0; i < phoneCount; i++) {
                Integer index = new Integer(i);
                mCi[i].registerForVirtualSimOn(this, EVENT_VIRTUAL_SIM_ON, index);
                mCi[i].registerForAvailable(this, EVENT_RADIO_AVAILABLE, null);
            }

        }
        mAirplaneRequestHandler = new AirplaneRequestHandler(mContext, mPhoneCount);
    }

    private int convertPhoneCountIntoBitmap(int phoneCount) {
        int ret = 0;
        for (int i = 0; i < phoneCount; i++) {
            ret += MODE_PHONE1_ONLY << i;
        }
        log("Convert phoneCount " + phoneCount + " into bitmap " + ret);
        return ret;
    }

    private BroadcastReceiver mIntentReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {

           log("BroadcastReceiver: " + intent.getAction());

            if (intent.getAction().equals(TelephonyIntents.ACTION_SIM_STATE_CHANGED)) {
                onReceiveSimStateChangedIntent(intent);
            } else if (intent.getAction().equals(ACTION_FORCE_SET_RADIO_POWER)) {
                onReceiveForceSetRadioPowerIntent(intent);
            } else if (intent.getAction().equals(ACTION_WIFI_ONLY_MODE_CHANGED)) {
                onReceiveWifiOnlyModeStateChangedIntent(intent);
            } else if (intent.getAction().equals(WifiManager.WIFI_STATE_CHANGED_ACTION)) {
                onReceiveWifiStateChangedIntent(intent);
            } else if (intent.getAction().equals(WIFI_OFFLOAD_SERVICE_ON)) {
                onReceiveWifiStateChangedIntent(intent);
            }
        }
    };

    protected void onReceiveWifiStateChangedIntent(Intent intent) {
        int extraWifiState = WifiManager.WIFI_STATE_DISABLED;
        if (intent.getAction().equals(WifiManager.WIFI_STATE_CHANGED_ACTION)) {
            log("Receiving WIFI_STATE_CHANGED_ACTION");
            extraWifiState = intent.getIntExtra(WifiManager.EXTRA_WIFI_STATE,
                                                WifiManager.WIFI_STATE_UNKNOWN);
        } else if (intent.getAction().equals(WIFI_OFFLOAD_SERVICE_ON)) {
            log("Receiving WIFI_OFFLOAD_SERVICE_ON");
            extraWifiState = intent.getBooleanExtra(WIFI_OFFLOAD_SERVICE_ON_EXTRA, false)
                             ? WifiManager.WIFI_STATE_ENABLED : WifiManager.WIFI_STATE_DISABLED;
        } else {
            log("Wrong intent");
            return;
        }

        log("airplaneMode: " + mAirplaneMode + " isFlightModePowerOffModem:"
                + isFlightModePowerOffModemConfigEnabled()
                + ", mIsWifiOn: " + mIsWifiOn);

        boolean modemPower;

        switch(extraWifiState) {
            case WifiManager.WIFI_STATE_ENABLED:
                log("WIFI_STATE_CHANGED enabled");
                modemPower = MODEM_POWER_ON;
                mIsWifiOn = true;
                if (mAirplaneMode == AIRPLANE_MODE_ON && isFlightModePowerOffModemConfigEnabled()) {
                    log("WIFI_STATE_CHANGED enabled, set modem on");
                    setSilentRebootPropertyForAllModem(IS_SILENT_REBOOT);
                    setModemPower(modemPower, mBitmapForPhoneCount);
                }
                break;
            case WifiManager.WIFI_STATE_DISABLED:
                log("WIFI_STATE_CHANGED disabled");
                modemPower = MODEM_POWER_OFF;
                mIsWifiOn = false;
                if (mAirplaneMode == AIRPLANE_MODE_ON && isFlightModePowerOffModemConfigEnabled()) {
                    log("WIFI_STATE_CHANGED disabled, set modem off");
                    setSilentRebootPropertyForAllModem(IS_SILENT_REBOOT);
                    setModemPower(modemPower, mBitmapForPhoneCount);
                }
                break;
            default:
                log("default: WIFI_STATE_CHANGED extra" + extraWifiState);
                break;
         }
    }

    protected void onReceiveSimStateChangedIntent(Intent intent) {
        String simStatus = intent.getStringExtra(IccCardConstants.INTENT_KEY_ICC_STATE);

        // TODO: phone_key now is equals to slot_key, change in the future
        int phoneId = intent.getIntExtra(PhoneConstants.PHONE_KEY, INVALID_PHONE_ID);

        if (!isValidPhoneId(phoneId)) {
            log("INTENT:Invalid phone id:" + phoneId + ", do nothing!");
            return;
        }

        log("INTENT:SIM_STATE_CHANGED: " + intent.getAction() + ", sim status: " + simStatus +
                ", phoneId: " + phoneId);

        boolean desiredRadioPower = RADIO_POWER_ON;

        if (IccCardConstants.INTENT_VALUE_ICC_READY.equals(simStatus)
            || IccCardConstants.INTENT_VALUE_ICC_LOCKED.equals(simStatus)
            || IccCardConstants.INTENT_VALUE_ICC_LOADED.equals(simStatus)) {
            mSimInsertedStatus[phoneId] = SIM_INSERTED;
            log("Phone[" + phoneId + "]: " + simStatusToString(SIM_INSERTED));

            // if we receive ready, but can't get iccid, we do nothing
            String iccid = readIccIdUsingPhoneId(phoneId);
            if (STRING_NO_SIM_INSERTED.equals(iccid)) {
                log("Phone " + phoneId + ":SIM ready but ICCID not ready, do nothing");
                return;
            }

            desiredRadioPower = RADIO_POWER_ON;
            if (mAirplaneMode == AIRPLANE_MODE_OFF) {
                log("Set Radio Power due to SIM_STATE_CHANGED, power: " + desiredRadioPower +
                        ", phoneId: " + phoneId);
                setRadioPower(desiredRadioPower, phoneId);
            }
        }

        else if (IccCardConstants.INTENT_VALUE_ICC_ABSENT.equals(simStatus)) {
            mSimInsertedStatus[phoneId] = NO_SIM_INSERTED;
            log("Phone[" + phoneId + "]: " + simStatusToString(NO_SIM_INSERTED));
            desiredRadioPower = RADIO_POWER_OFF;
            if (mAirplaneMode == AIRPLANE_MODE_OFF) {
                log("Set Radio Power due to SIM_STATE_CHANGED, power: " + desiredRadioPower +
                        ", phoneId: " + phoneId);
                setRadioPower(desiredRadioPower, phoneId);
            }
        }
    }

   /**
     * enter or leave wifi only mode
     *
     */
    public void onReceiveWifiOnlyModeStateChangedIntent(Intent intent) {

        boolean enabled = intent.getBooleanExtra("state", false);
        log("mReceiver: ACTION_WIFI_ONLY_MODE_CHANGED, enabled = " + enabled);

        // we expect wifi only mode is on-> off or off->on
        if (enabled == mWifiOnlyMode) {
            log("enabled = " + enabled + ", mWifiOnlyMode = "+ mWifiOnlyMode +
                "is not expected (the same)");
            return;
        }

        mWifiOnlyMode = enabled;
        if (mAirplaneMode == AIRPLANE_MODE_OFF) {
            boolean radioPower = enabled ? RADIO_POWER_OFF : RADIO_POWER_ON;
            for (int i = 0; i < mPhoneCount; i++) {
                setRadioPower(radioPower, i);
            }
        }
    }

    private void onReceiveForceSetRadioPowerIntent(Intent intent) {
        int phoneId = 0;
        int mode = intent.getIntExtra(Intent.EXTRA_MSIM_MODE, -1);
        log("force set radio power, mode: " + mode);
        if (mode == -1) {
            log("Invalid mode, MSIM_MODE intent has no extra value");
            return;
        }
        for (phoneId = 0; phoneId < mPhoneCount; phoneId++) {
            boolean singlePhonePower =
                ((mode & (MODE_PHONE1_ONLY << phoneId)) == 0) ? RADIO_POWER_OFF : RADIO_POWER_ON;
            if (RADIO_POWER_ON == singlePhonePower) {
                forceSetRadioPower(true, phoneId);
            }
        }
    }

    protected boolean isValidPhoneId(int phoneId) {
        if (phoneId < 0 || phoneId >= TelephonyManager.getDefault().getPhoneCount()) {
            return false;
        } else {
            return true;
        }
    }

    protected String simStatusToString(int simStatus) {
        String result = null;
        switch(simStatus) {
            case SIM_NOT_INITIALIZED:
                result = "SIM HAVE NOT INITIALIZED";
                break;
            case SIM_INSERTED:
                result = "SIM DETECTED";
                break;
            case NO_SIM_INSERTED:
                result = "NO SIM DETECTED";
                break;
            }
        return result;
    }

    /**
     * Modify mAirplaneMode and set modem power
     * @param enabled 0: normal mode
     *                1: airplane mode
     * @internal
     */
    public void notifyAirplaneModeChange(boolean enabled) {
        ///M: Add for the airplane mode frequently switch issue.@{
        if (!mAirplaneRequestHandler.allowSwitching()) {
            log("airplane mode switching, not allow switch now ");
            mAirplaneRequestHandler.pendingAirplaneModeRequest(enabled);
            return;
        }
        if (mAirplaneRequestHandler.waitForReady(enabled)) {
            log("airplane mode switching, wait for ready, not allow switch now");
            return;
        }
        /// @}

        // we expect airplane mode is on-> off or off->on
        if (enabled == mAirplaneMode) {
            log("enabled = " + enabled + ", mAirplaneMode = " + mAirplaneMode +
                    "is not expected (the same)");
            return;
        }

        mAirplaneMode = enabled;
        log("Airplane mode changed:" + enabled);
        SystemProperties.set("persist.radio.airplane.mode.on", enabled ? "true" : "false");

        /// Handle WFC and Airplane mode special case @{
        boolean currModemPower = true;
        if (isModemPowerOff(0)) {  // phoneId is not used for non GSM+GSM DSDA project
            currModemPower = false;
        }
        int radioAction = -1;  // 0:skip, 1:setRadioPower, 2:setModemPower
        /// @}

        if (isFlightModePowerOffModemConfigEnabled() && !isUnderCryptKeeper()) {
            /// Handle WFC and Airplane mode special case @{
            // Wifi ON, current Modem ON => set Radio Power according to airplane mode
            if (mIsWifiOn && currModemPower) {
                log("Airplane mode changed: turn on/off all radio due to WFC on");
                radioAction = TO_SET_RADIO_POWER;
            // Wifi OFF, current Modem ON, target Airplane mode OFF => conflict, set Radio Power
            } else if (!mIsWifiOn && currModemPower && mAirplaneMode == AIRPLANE_MODE_OFF) {
                log("Airplane mode changed: turn on all radio due to mode conflict & WFC off");
                radioAction = TO_SET_RADIO_POWER;
            // Wifi OFF, current Modem OFF, target Airplane mode ON => conflict, skip
            } else if (!mIsWifiOn && !currModemPower && mAirplaneMode == AIRPLANE_MODE_ON) {
                log("Airplane mode changed: skip & fake DONE due to mode conflict & WFC off");
                radioAction = TO_SKIP_AND_FAKE_DONE;
            /// @}
            } else {
                log("Airplane mode changed: turn on/off all modem");
                radioAction = TO_SET_MODEM_POWER;
            }
        } else if (isMSimModeSupport()) {
            log("Airplane mode changed: turn on/off all radio");
            radioAction = TO_SET_RADIO_POWER;
        }

        // MD power off flag @{
        if (SystemProperties.get(PROPERTY_RIL_MD_OFF).equals("true")) {
            log("PROPERTY_RIL_MD_OFF == true");
            radioAction = TO_SET_MODEM_POWER;
        }
        // @})

        if (radioAction == TO_SKIP_AND_FAKE_DONE) {
            Intent intent = new Intent(ACTION_AIRPLANE_CHANGE_DONE);
            intent.putExtra(EXTRA_AIRPLANE_MODE, false);
            mContext.sendBroadcastAsUser(intent, UserHandle.ALL);
        } else if (radioAction == TO_SET_RADIO_POWER) {
            boolean radioPower = enabled ? RADIO_POWER_OFF : RADIO_POWER_ON;
            for (int i = 0; i < mPhoneCount; i++) {
                setRadioPower(radioPower, i);
            }
            ///M: Add for the airplane mode frequently switch issue.@{
            mAirplaneRequestHandler.monitorRadioChangeDone(radioPower);
            /// @}
        } else if (radioAction == TO_SET_MODEM_POWER) {
            boolean modemPower = enabled ? MODEM_POWER_OFF : MODEM_POWER_ON;
            setSilentRebootPropertyForAllModem(IS_SILENT_REBOOT);
            setModemPower(modemPower, mBitmapForPhoneCount);
        }
    }

    /*
     *A special paragraph, not to trun off modem power under cryptkeeper
     */
    protected boolean isUnderCryptKeeper() {
        /*
        /* android N provides new encryption  mechanism "file base encryption"
        /* FBS would not enctypt NV-Ram of modem. So, RadioManager can turn on/off
        /* under FBE
        /* ro.crypto.type == block => FDE
        /* ro.crypto.type == file => FBE
        */
        if (SystemProperties.get("ro.crypto.type").equals("block")
            && SystemProperties.get("ro.crypto.state").equals("encrypted")
            && SystemProperties.get("vold.decrypt").equals("trigger_restart_min_framework")) {
            log("[Special Case] Under CryptKeeper, Not to turn on/off modem");
            return true;
        }
        log("[Special Case] Not Under CryptKeeper");
        return false;
    }

    public void setSilentRebootPropertyForAllModem(String isSilentReboot) {
        TelephonyManager.MultiSimVariants config =
                TelephonyManager.getDefault().getMultiSimConfiguration();
        switch(config) {
            case DSDS:
                log("set eboot under DSDS");
                SystemProperties.set(PROPERTY_SILENT_REBOOT_MD1, isSilentReboot);
                break;
            case DSDA:
                log("set eboot under DSDA");
                SystemProperties.set(PROPERTY_SILENT_REBOOT_MD1, isSilentReboot);
                SystemProperties.set(PROPERTY_SILENT_REBOOT_MD2, isSilentReboot);
                break;
            case TSTS:
                log("set eboot under TSTS");
                SystemProperties.set(PROPERTY_SILENT_REBOOT_MD1, isSilentReboot);
                break;
            default:
                log("set eboot under SS");
                SystemProperties.set(PROPERTY_SILENT_REBOOT_MD1, isSilentReboot);
                break;
        }
        // Check CDMA
        if (SystemProperties.get(MTK_C2K_SUPPORT).equals("1")) {
            log("C2K project, set cdma.ril.eboot to " + isSilentReboot);
            SystemProperties.set(PROPERTY_SILENT_REBOOT_CDMA, isSilentReboot);
        }
    }

    /*
     * Called From GSMSST, if boot up under airplane mode, power-off modem
     */
    public void notifyRadioAvailable(int phoneId) {
        log("Phone " + phoneId + " notifies radio available");
        log("RADIO_AVAILABLE: airplane mode: " + mAirplaneMode + " cryptkeeper: "
             + isUnderCryptKeeper() + " mIsWifiOn:" + mIsWifiOn);

        if (mAirplaneMode == AIRPLANE_MODE_ON && isFlightModePowerOffModemConfigEnabled()
                && !isUnderCryptKeeper() && !mIsWifiOn) {
            log("Power off modem because boot up under airplane mode");
            setModemPower(MODEM_POWER_OFF, MODE_PHONE1_ONLY << phoneId);
        }
    }

    public void notifyIpoShutDown() {
        log("notify IPO shutdown!");
        bIsInIpoShutdown = true;
        bIsQueueIpoPreboot = false;

        for (int i = 0; i < TelephonyManager.getDefault().getPhoneCount(); i++) {
            // record IpoShutdown if there is any phone of radio state is not avaible,
            // then do ipo shutdown after radio available
            log("mCi[" + i + "].getRadioState().isAvailable(): " +
                mCi[i].getRadioState().isAvailable());
            if (!mCi[i].getRadioState().isAvailable()) {
                bIsQueueIpoShutdown = true;
            }
        }

        // it may fail on the phone which radio state is not available
        setModemPower(MODEM_POWER_OFF, mBitmapForPhoneCount);
    }

    public void notifyIpoPreBoot() {
        log("IPO preboot!");
        bIsInIpoShutdown = false;
        bIsQueueIpoShutdown = false;
        setSilentRebootPropertyForAllModem(IS_NOT_SILENT_REBOOT);

        for (int i = 0; i < TelephonyManager.getDefault().getPhoneCount(); i++) {
            // record IpoPreboot if there is any phone of radio state is not avaible,
            // then do ipo preboot after radio available
            log("mCi[" + i + "].getRadioState().isAvailable(): " +
                mCi[i].getRadioState().isAvailable());
            if (!mCi[i].getRadioState().isAvailable()) {
                bIsQueueIpoPreboot = true;
            }
        }

        setModemPower(MODEM_POWER_ON, mBitmapForPhoneCount);
    }

    /**
     * Set modem power on/off according to DSDS or DSDA.
     *
     * @param power desired power of modem
     * @param phoneId a bit map of phones you want to set
     *              1: phone 1 only
     *              2: phone 2 only
     *              3: phone 1 and phone 2
     */
    public void setModemPower(boolean power, int phoneBitMap) {
        log("Set Modem Power according to bitmap, Power:" + power + ", PhoneBitMap:" + phoneBitMap);
        TelephonyManager.MultiSimVariants config =
                TelephonyManager.getDefault().getMultiSimConfiguration();

        final Message[] responses;
        ///M: Add for the airplane mode frequently switch issue.@{
        responses = mAirplaneRequestHandler.monitorModemPowerChangeDone(
                power, phoneBitMap, findMainCapabilityPhoneId());
        /// @}

        int phoneId = 0;
        switch(config) {
            case DSDS:
                phoneId = findMainCapabilityPhoneId();
                log("Set Modem Power under DSDS mode, Power:" + power + ", phoneId:" + phoneId);
                log("PROPERTY_RIL_MD_OFF = " + SystemProperties.get(PROPERTY_RIL_MD_OFF));
                mCi[phoneId].setModemPower(power, responses[phoneId]);
                // MD power off flag @{
                if (power == MODEM_POWER_OFF) {
                    SystemProperties.set(PROPERTY_RIL_MD_OFF, "true");
                    for (int i = 0; i < mPhoneCount; i++) {
                        resetSimInsertedStatus(i);
                    }
                } else {
                    SystemProperties.set(PROPERTY_RIL_MD_OFF, "false");
                }
                // @}
                break;

            case DSDA:
                for (int i = 0; i < mPhoneCount; i++) {
                    phoneId = i;
                    if ((phoneBitMap & (MODE_PHONE1_ONLY << i)) != 0) {
                        log("Set Modem Power under DSDA mode, Power:" + power + ", phoneId:" +
                                phoneId);
                        log("PROPERTY_RIL_MD_OFF = " + SystemProperties.get(PROPERTY_RIL_MD_OFF));
                        mCi[phoneId].setModemPower(power, responses[phoneId]);
                        // MD power off flag @{
                        if (power == MODEM_POWER_OFF) {
                            SystemProperties.set(PROPERTY_RIL_MD_OFF, "true");
                            resetSimInsertedStatus(phoneId);
                        } else {
                            SystemProperties.set(PROPERTY_RIL_MD_OFF, "false");
                        }
                        // @}
                    }
                }
                break;

            case TSTS:
                phoneId = findMainCapabilityPhoneId();
                log("Set Modem Power under TSTS mode, Power:" + power + ", phoneId:" + phoneId);
                log("PROPERTY_RIL_MD_OFF = " + SystemProperties.get(PROPERTY_RIL_MD_OFF));
                mCi[phoneId].setModemPower(power, responses[phoneId]);
                // MD power off flag @{
                if (power == MODEM_POWER_OFF) {
                    SystemProperties.set(PROPERTY_RIL_MD_OFF, "true");
                    for (int i = 0; i < mPhoneCount; i++) {
                        resetSimInsertedStatus(i);
                    }
                } else {
                    SystemProperties.set(PROPERTY_RIL_MD_OFF, "false");
                }
                // @}
                break;

            default:
                phoneId = PhoneFactory.getDefaultPhone().getPhoneId();
                log("Set Modem Power under SS mode:" + power + ", phoneId:" + phoneId);
                log("PROPERTY_RIL_MD_OFF = " + SystemProperties.get(PROPERTY_RIL_MD_OFF));
                mCi[phoneId].setModemPower(power, responses[phoneId]);
                // MD power off flag @{
                if (power == MODEM_POWER_OFF) {
                    SystemProperties.set(PROPERTY_RIL_MD_OFF, "true");
                } else {
                    SystemProperties.set(PROPERTY_RIL_MD_OFF, "false");
                }
                // @}
                break;
        }
    }

    protected int findMainCapabilityPhoneId() {
        int result = 0;
        int switchStatus = Integer.valueOf(
                SystemProperties.get(PhoneConstants.PROPERTY_CAPABILITY_SWITCH, "1"));
        result = switchStatus - 1;
        if (result < 0 || result >= mPhoneCount) {
            return 0;
        } else {
            return result;
        }
    }

    /**
     * Radio power Runnable for power on/off retry.
     */
    protected class RadioPowerRunnable implements Runnable {
        boolean retryPower;
        int retryPhoneId;
        public  RadioPowerRunnable(boolean power, int phoneId) {
            retryPower = power;
            retryPhoneId = phoneId;
        }
        @Override
        public void run() {
            setRadioPower(retryPower, retryPhoneId);
        }
    }

    /*
     * MTK flow to control radio power
     */
    public void setRadioPower(boolean power, int phoneId) {
        log("setRadioPower, power=" + power + "  phoneId=" + phoneId);
        Phone phone;
        phone = PhoneFactory.getPhone(phoneId);
        if (phone == null) {
            return;
        }

        if ((isFlightModePowerOffModemEnabled() || power) && mAirplaneMode == AIRPLANE_MODE_ON) {
            log("Set Radio Power on under airplane mode, ignore");
            return;
        }

        ///M:There is no modem on wifi-only device @{
        ConnectivityManager cm = (ConnectivityManager) mContext.getSystemService(
                Context.CONNECTIVITY_SERVICE);
        if (cm.isNetworkSupported(ConnectivityManager.TYPE_MOBILE) == false) {
            log("wifi-only device, so return");
            return;
        }
        ///  @}

        if (isModemPowerOff(phoneId)) {
            log("modem for phone " + phoneId + " off, do not set radio again");
            return;
        }

        /**
        * We want iccid ready berfore we check if SIM is once manually turned-offedd
        * So we check ICCID repeatedly every 300 ms
        */
        if (!isIccIdReady(phoneId)) {
            log("RILD initialize not completed, wait for " + INITIAL_RETRY_INTERVAL_MSEC + "ms");
            RadioPowerRunnable setRadioPowerRunnable = new RadioPowerRunnable(power, phoneId);
            postDelayed(setRadioPowerRunnable, INITIAL_RETRY_INTERVAL_MSEC);
            return;
        }

        setSimInsertedStatus(phoneId);

        boolean radioPower = power;
        String iccId = readIccIdUsingPhoneId(phoneId);
        //adjust radio power according to ICCID
        if (sIccidPreference.contains(iccId)) {
            log("Adjust radio to off because once manually turned off, iccid: " + iccId +
                    " , phone: " + phoneId);
            radioPower = RADIO_POWER_OFF;
        }

        if (mWifiOnlyMode == WIFI_ONLY_MODE_ON && mIsEccCall == false) {
            log("setradiopower but wifi only, turn off");
            radioPower = RADIO_POWER_OFF;
        }

        boolean isCTACase = checkForCTACase();

        if (getSimInsertedStatus(phoneId) == NO_SIM_INSERTED) {
            if (isCTACase == true) {
                int capabilityPhoneId = findMainCapabilityPhoneId();
                log("No SIM inserted, force to turn on 3G/4G phone " +
                    capabilityPhoneId + " radio if no any sim radio is enabled!");
                PhoneFactory.getPhone(capabilityPhoneId).setRadioPower(RADIO_POWER_ON);
                for (int i = 0; i < mPhoneCount; i++) {
                    phone = PhoneFactory.getPhone(i);
                    if (phone == null) {
                        continue;
                    }
                    if (i != capabilityPhoneId && mIsEccCall != true) {
                        phone.setRadioPower(RADIO_POWER_OFF);
                    }
                }

            } else if (true == mIsEccCall) {
                log("ECC call Radio Power, power: " + radioPower + ", phoneId: " + phoneId);
                PhoneFactory.getPhone(phoneId).setRadioPower(radioPower);
            } else {
                log("No SIM inserted, turn Radio off!");
                radioPower = RADIO_POWER_OFF;
                PhoneFactory.getPhone(phoneId).setRadioPower(radioPower);
            }
        } else {
            log("Trigger set Radio Power, power: " + radioPower + ", phoneId: " + phoneId);
            // We must refresh sim setting during boot up or if we adjust power according to ICCID
            refreshSimSetting(radioPower, phoneId);
            PhoneFactory.getPhone(phoneId).setRadioPower(radioPower);
        }
    }

    protected int getSimInsertedStatus(int phoneId) {
        return mSimInsertedStatus[phoneId];
    }

    protected void setSimInsertedStatus(int phoneId) {
        String iccId = readIccIdUsingPhoneId(phoneId);
        if (STRING_NO_SIM_INSERTED.equals(iccId)) {
            mSimInsertedStatus[phoneId] = NO_SIM_INSERTED;
        } else {
            mSimInsertedStatus[phoneId] = SIM_INSERTED;
        }
    }

    protected boolean isIccIdReady(int phoneId) {
        String iccId = readIccIdUsingPhoneId(phoneId);
        boolean ret = ICC_READ_NOT_READY;
        if (iccId == null || "".equals(iccId)) {
            ret = ICC_READ_NOT_READY;
        } else {
            //log("ICC read ready, iccid[" + phoneId + "]: " + iccId);
            ret = ICC_READ_READY;
        }
        return ret;
    }

    protected String readIccIdUsingPhoneId(int phoneId) {
        String ret = SystemProperties.get(PROPERTY_ICCID_SIM[phoneId]);
        log("ICCID for phone " + phoneId + " is " + ret);
        return ret;
    }

    protected boolean checkForCTACase() {
        boolean isCTACase = true;
        //log("Check For CTA case!");
        if (mAirplaneMode == AIRPLANE_MODE_OFF && mWifiOnlyMode != WIFI_ONLY_MODE_ON){
            for (int i = 0; i < mPhoneCount; i++) {
                log("Check For CTA case: mSimInsertedStatus[" + i + "]:"  + mSimInsertedStatus[i]);
                if (mSimInsertedStatus[i] == SIM_INSERTED ||
                        mSimInsertedStatus[i] == SIM_NOT_INITIALIZED) {
                    isCTACase = false;
                }
            }
        } else {
            isCTACase = false;
        }

        if ((false == isCTACase) && (false == mIsEccCall)) {
            turnOffCTARadioIfNecessary();
        }
        log("CTA case: " + isCTACase);
        return isCTACase;
    }

    /*
     * We need to turn off Phone's radio if no SIM inserted (radio on because CTA) after we leave
     * CTA case
     */
    private void turnOffCTARadioIfNecessary() {
        Phone phone;
        for (int i = 0; i < mPhoneCount; i++) {
            phone = PhoneFactory.getPhone(i);
            if (phone == null) {
                continue;
            }
            if (mSimInsertedStatus[i] == NO_SIM_INSERTED) {
                if (isModemPowerOff(i)) {
                    log("modem off, not to handle CTA");
                    return;
                } else {
                    log("turn off phone " + i + " radio because we are no longer in CTA mode");
                    phone.setRadioPower(RADIO_POWER_OFF);
                }
            }
        }
    }

    /*
     * Refresh MSIM Settings only when:
     * We auto turn off a SIM card once manually turned off
     */
    protected void refreshSimSetting(boolean radioPower, int phoneId) {
        int simMode = Settings.Global.getInt(mContext.getContentResolver(),
                          Settings.Global.MSIM_MODE_SETTING, mBitmapForPhoneCount);
        int oldMode = simMode;

        if (radioPower == RADIO_POWER_OFF) {
            simMode &= ~(MODE_PHONE1_ONLY << phoneId);
        } else {
            simMode |= (MODE_PHONE1_ONLY << phoneId);
        }

        if (simMode != oldMode) {
            log("Refresh MSIM mode setting to " + simMode + " from " + oldMode);
            Settings.Global.putInt(mContext.getContentResolver(),
                Settings.Global.MSIM_MODE_SETTING, simMode);
        }
    }

    /**
     * Wait ICCID ready when force set radio power.
     */
    protected class ForceSetRadioPowerRunnable implements Runnable {
        boolean mRetryPower;
        int mRetryPhoneId;
        public  ForceSetRadioPowerRunnable(boolean power, int phoneId) {
            mRetryPower = power;
            mRetryPhoneId = phoneId;
        }
        @Override
        public void run() {
            forceSetRadioPower(mRetryPower, mRetryPhoneId);
        }
    }

    /**
     * force turn on radio and remove iccid for preference to prevent being turned off again
     * 1. For ECC call
     */
    public void forceSetRadioPower(boolean power, int phoneId) {
        log("force set radio power for phone" + phoneId + " ,power: " + power);
        Phone phone;
        phone = PhoneFactory.getPhone(phoneId);
        if (phone == null) {
            return;
        }

        if (isFlightModePowerOffModemConfigEnabled() && mAirplaneMode == AIRPLANE_MODE_ON) {
            log("Force Set Radio Power under airplane mode, ignore");
            return;
        }

        if (bIsInIpoShutdown) {
            log("Force Set Radio Power under ipo shutdown, ignore");
            return;
        }

        if (isModemPowerOff(phoneId) && mAirplaneMode == AIRPLANE_MODE_ON) {
            // refreshIccIdPreference(power, readIccIdUsingPhoneId(phoneId));
            log("Modem Power Off for phone " + phoneId + ", Power on modem first");
            setModemPower(MODEM_POWER_ON, MODE_PHONE1_ONLY << phoneId);
            // return;
        }

        /**
        * We want iccid ready berfore we check if SIM is once manually turned-offedd
        * So we check ICCID repeatedly every 300 ms
        */
        if (!isIccIdReady(phoneId)) {
            log("force set radio power, read iccid not ready, wait for" +
                INITIAL_RETRY_INTERVAL_MSEC + "ms");
            ForceSetRadioPowerRunnable forceSetRadioPowerRunnable =
                new ForceSetRadioPowerRunnable(power, phoneId);
            postDelayed(forceSetRadioPowerRunnable,
                INITIAL_RETRY_INTERVAL_MSEC);
            return;
        }

        boolean radioPower = power;
        refreshIccIdPreference(radioPower, readIccIdUsingPhoneId(phoneId));
        phone.setRadioPower(power);
    }

    /**
     * Force turn on radio and remove iccid for preference to prevent being turned off again
     * For CT ECC call
     * @param power for on/off radio power
     * @param phoneId for phone ID
     * @param isEccOn for if ECC call on-going
     */
    public void forceSetRadioPower(boolean power, int phoneId, boolean isEccOn) {
        log("force set radio power isEccOn: " + isEccOn);
        mIsEccCall = isEccOn;
        forceSetRadioPower(power, phoneId);
    }

    /*
     * wait ICCID ready when SIM mode change
     */
    private class SimModeChangeRunnable implements Runnable {
        boolean mPower;
        int mPhoneId;
        public SimModeChangeRunnable(boolean power, int phoneId) {
            mPower = power;
            mPhoneId = phoneId;
        }
        @Override
        public void run() {
            notifySimModeChange(mPower, mPhoneId);
        }
    }

    /**
     * Refresh ICCID preference due to toggling on SIM management except for below cases:
     * 1. SIM Mode Feature not defined
     * 2. Under Airplane Mode (PhoneGlobals will call GSMPhone.setRadioPower after receving
     * airplane mode change)
     * @param power power on -> remove preference
     *               power off -> add to preference
     */
    public void notifySimModeChange(boolean power, int phoneId) {
        log("SIM mode changed, power: " + power + ", phoneId" + phoneId);
        if (!isMSimModeSupport() || mAirplaneMode == AIRPLANE_MODE_ON) {
            log("Airplane mode on or MSIM Mode option is closed, do nothing!");
            return;
        } else {
            if (!isIccIdReady(phoneId)) {
                log("sim mode read iccid not ready, wait for "
                    + INITIAL_RETRY_INTERVAL_MSEC + "ms");
                SimModeChangeRunnable notifySimModeChangeRunnable
                    = new SimModeChangeRunnable(power, phoneId);
                postDelayed(notifySimModeChangeRunnable, INITIAL_RETRY_INTERVAL_MSEC);
                return;
            }
            //once ICCIDs are ready, then set the radio power
            if (STRING_NO_SIM_INSERTED.equals(readIccIdUsingPhoneId(phoneId))) {
                power = RADIO_POWER_OFF;
                log("phoneId " + phoneId + " sim not insert, set  power  to " + power);
            }
            refreshIccIdPreference(power, readIccIdUsingPhoneId(phoneId));
            log("Set Radio Power due to SIM mode change, power: " + power + ", phoneId: "
                    + phoneId);
            setPhoneRadioPower(power, phoneId);
        }
    }

    protected void setPhoneRadioPower(boolean power, int phoneId) {
        Phone phone;
        phone = PhoneFactory.getPhone(phoneId);
        if (phone == null) {
            return;
        }
        phone.setRadioPower(power);
    }


    /**
     * Wait ICCID ready when MSIM modem change.
     * @Deprecated
     */
    protected class MSimModeChangeRunnable implements Runnable {
        int mRetryMode;
        public  MSimModeChangeRunnable(int mode) {
            mRetryMode = mode;
        }
        @Override
        public void run() {
            notifyMSimModeChange(mRetryMode);
        }
    }

    /**
     * Refresh ICCID preference due to toggling on SIM management except for below cases:
     * 1. SIM Mode Feature not defined
     * 2. Under Airplane Mode (PhoneGlobals will call GSMPhone.setRadioPower after receving
     * airplane mode change)
     * @param power power on -> remove preference
     *               power off -> add to preference
     * @internal
     * @Deprecated
     */
    public void notifyMSimModeChange(int mode) {
        log("MSIM mode changed, mode: " + mode);
        if (mode == -1) {
            log("Invalid mode, MSIM_MODE intent has no extra value");
            return;
        }
        if (!isMSimModeSupport() || mAirplaneMode == AIRPLANE_MODE_ON) {
            log("Airplane mode on or MSIM Mode option is closed, do nothing!");
            return;
        } else {
            //all ICCCIDs need be ready berfore set radio power
            int phoneId = 0;
            boolean iccIdReady = true;
            for (phoneId = 0; phoneId < mPhoneCount; phoneId++) {
                if (!isIccIdReady(phoneId)) {
                    iccIdReady = false;
                    break;
                }
            }

            removeCallbacks(mNotifyMSimModeChangeRunnable);
            if (!iccIdReady) {
                /*log("msim mode read iccid not ready, wait for "
                    + INITIAL_RETRY_INTERVAL_MSEC + "ms");*/
                mNotifyMSimModeChangeRunnable
                    = new MSimModeChangeRunnable(mode);
                postDelayed(mNotifyMSimModeChangeRunnable, INITIAL_RETRY_INTERVAL_MSEC);
                return;
            }
            //once ICCIDs are ready, then set the radio power
            for (phoneId = 0; phoneId < mPhoneCount; phoneId++) {
                boolean singlePhonePower = ((mode & (MODE_PHONE1_ONLY << phoneId)) == 0) ?
                        RADIO_POWER_OFF : RADIO_POWER_ON;
                if (STRING_NO_SIM_INSERTED.equals(readIccIdUsingPhoneId(phoneId))) {
                    singlePhonePower = RADIO_POWER_OFF;
                    log("phoneId " + phoneId + " sim not insert, set  power  to "
                            + singlePhonePower);
                }
                refreshIccIdPreference(singlePhonePower, readIccIdUsingPhoneId(phoneId));
                log("Set Radio Power due to MSIM mode change, power: " + singlePhonePower
                        + ", phoneId: " + phoneId);
                setPhoneRadioPower(singlePhonePower, phoneId);
            }
        }
    }

    protected void refreshIccIdPreference(boolean power, String iccid) {
        log("refresh iccid preference");
        SharedPreferences.Editor editor = sIccidPreference.edit();
        if (power == RADIO_POWER_OFF && !STRING_NO_SIM_INSERTED.equals(iccid)) {
            putIccIdToPreference(editor, iccid);
        } else {
            removeIccIdFromPreference(editor, iccid);
        }
        editor.commit();
    }

    private void putIccIdToPreference(SharedPreferences.Editor editor, String iccid) {
        if (iccid != null) {
            log("Add radio off SIM: " + iccid);
            editor.putInt(iccid, 0);
         }
    }

    private void removeIccIdFromPreference(SharedPreferences.Editor editor, String iccid) {
        if (iccid != null) {
            log("Remove radio off SIM: " + iccid);
            editor.remove(iccid);
        }
    }

    /*
     * Some Request or AT command must made before EFUN
     * 1. Prevent waiting for response
     * 2. Send commands as the same channel as EFUN or CFUN
     */
    public static void sendRequestBeforeSetRadioPower(boolean power, int phoneId) {
        log("Send request before EFUN, power:" + power + " phoneId:" + phoneId);

        notifyRadioPowerChange(power, phoneId);
    }

    /**
     * MTK Power on feature
     * 1. Radio off a card from SIM Management
     * 2. Flight power off modem
     * @internal
     */
    public static boolean isPowerOnFeatureAllClosed() {
        boolean ret = true;
        if (isFlightModePowerOffModemConfigEnabled()) {
            ret = false;
        } else if (isRadioOffPowerOffModemEnabled()) {
            ret = false;
        } else if (isMSimModeSupport()) {
            ret = false;
        }
        return ret;
    }

    public static boolean isRadioOffPowerOffModemEnabled() {
        return SystemProperties.get("ro.mtk_radiooff_power_off_md").equals("1");
    }

    public static boolean isFlightModePowerOffModemConfigEnabled() {
        //first check testmode
        if (SystemProperties.get("ril.testmode").equals("1")) {
            return SystemProperties.get("ril.test.poweroffmd").equals("1");
        }

        //second check special test request
        //CMCC VOLTE TEST NEED START
        String optr = SystemProperties.get("persist.operator.optr");
        boolean isOP01 = (optr != null) && optr.equalsIgnoreCase("op01");
        boolean isTestSim = SystemProperties.get("gsm.sim.ril.testsim").equals("1") ||
                   SystemProperties.get("gsm.sim.ril.testsim.2").equals("1") ||
                   SystemProperties.get("gsm.sim.ril.testsim.3").equals("1") ||
                   SystemProperties.get("gsm.sim.ril.testsim.4").equals("1");
        if (isOP01 && isTestSim) {
            return true;
        }
        boolean hasIccCard = TelephonyManager.getDefault().hasIccCard(0) ||
            TelephonyManager.getDefault().hasIccCard(1);
        //String operatorNumeric = PhoneFactory.getPhone(0).getServiceStateTracker().getLocatedPlmn();
        log("isFlightModePowerOffModemEnabled: hasIccCard: " + hasIccCard);
        if (isOP01 && !hasIccCard)
            return true;

        //at last check the project feature option.
        return SystemProperties.get("ro.mtk_flight_mode_power_off_md").equals("1");
    }

    public static boolean isFlightModePowerOffModemEnabled() {

        if (RadioManager.getInstance() == null) {
            log("Instance not exists, return config only");
            return isFlightModePowerOffModemConfigEnabled();
        }

        if (isFlightModePowerOffModemConfigEnabled() == true) {
            return !RadioManager.getInstance().mIsWifiOn;
        } else {
            return false;
        }
    }

    /**
     *  Check if modem is already power off.
     **/
    public static boolean isModemPowerOff(int phoneId) {
        return RadioManager.getInstance().isModemOff(phoneId);
    }

    public static boolean isMSimModeSupport() {
        // TODO: adds logic
        if (SystemProperties.get("ro.mtk_bsp_package").equals("1")) {
            return false;
        } else {
            return true;
        }
    }

    private void setAirplaneMode(boolean enabled) {
        log("set mAirplaneMode as:" + enabled);
        mAirplaneMode = enabled;
    }

    private boolean getAirplaneMode() {
        return mAirplaneMode;
    }


    protected void resetSimInsertedStatus(int phoneId) {
        log("reset Sim InsertedStatus for Phone:" + phoneId);
        mSimInsertedStatus[phoneId] = SIM_NOT_INITIALIZED;
    }

    @Override
    public void handleMessage(Message msg) {
        AsyncResult ar;
        int[] ints;
        String[] strings;
        Message message;
        int phoneIdForMsg = getCiIndex(msg);

        log("handleMessage msg.what: " + eventIdtoString(msg.what));
        switch (msg.what) {
            case EVENT_RADIO_AVAILABLE:
                if (bIsQueueIpoShutdown) {
                    log("bIsQueueIpoShutdown is true");
                    bIsQueueIpoShutdown = false;
                    // Retry IPO shutdown when radio available
                    log("IPO shut down retry!");
                    setModemPower(MODEM_POWER_OFF, mBitmapForPhoneCount);
                }
                if (bIsQueueIpoPreboot) {
                    log("bIsQueueIpoPreboot is true");
                    bIsQueueIpoPreboot = false;
                    // Retry IPO preboot when radio available
                    log("IPO reboot retry!");
                    setModemPower(MODEM_POWER_ON, mBitmapForPhoneCount);
                }
                break;
            case EVENT_VIRTUAL_SIM_ON:
                forceSetRadioPower(RADIO_POWER_ON, phoneIdForMsg);
                break;
            default:
                super.handleMessage(msg);
                break;
        }
    }

    private String eventIdtoString(int what) {
        String str = null;
        switch (what) {
            case EVENT_RADIO_AVAILABLE:
                str = "EVENT_RADIO_AVAILABLE";
                break;
            case EVENT_VIRTUAL_SIM_ON:
                str = "EVENT_VIRTUAL_SIM_ON";
                break;
            default:
                break;
        }
        return str;
    }

    private int getCiIndex(Message msg) {
        AsyncResult ar;
        Integer index = new Integer(PhoneConstants.DEFAULT_CARD_INDEX);

        /*
         * The events can be come in two ways. By explicitly sending it using
         * sendMessage, in this case the user object passed is msg.obj and from
         * the CommandsInterface, in this case the user object is msg.obj.userObj
         */
        if (msg != null) {
            if (msg.obj != null && msg.obj instanceof Integer) {
                index = (Integer) msg.obj;
            } else if (msg.obj != null && msg.obj instanceof AsyncResult) {
                ar = (AsyncResult) msg.obj;
                if (ar.userObj != null && ar.userObj instanceof Integer) {
                    index = (Integer) ar.userObj;
                }
            }
        }
        return index.intValue();
    }

    protected boolean isModemOff(int phoneId) {
        boolean powerOff = false;
        TelephonyManager.MultiSimVariants config
            = TelephonyManager.getDefault().getMultiSimConfiguration();
        switch(config) {
            case DSDS:
                powerOff = !SystemProperties.get("ril.ipo.radiooff").equals("0");
                break;
            case DSDA:
                switch (phoneId) {
                    case 0: //phone 1
                        powerOff = !SystemProperties.get("ril.ipo.radiooff").equals("0");
                        break;
                    case 1: //phone 2
                        powerOff = !SystemProperties.get("ril.ipo.radiooff.2").equals("0");
                        break;
                    default:
                        powerOff = true;
                        break;
                }
                break;
            case TSTS:
                //TODO: check 3 SIM case
                powerOff = !SystemProperties.get("ril.ipo.radiooff").equals("0");
                break;
            default:
                 powerOff = !SystemProperties.get("ril.ipo.radiooff").equals("0");
                break;
        }
        return powerOff;
    }

     public static synchronized void registerForRadioPowerChange(String name,
            IRadioPower iRadioPower) {
        if (name == null) {
            name = REGISTRANTS_WITH_NO_NAME;
        }
        log(name + " registerForRadioPowerChange");
        mNotifyRadioPowerChange.put(iRadioPower, name);
    }

    public static synchronized void unregisterForRadioPowerChange(IRadioPower iRadioPower) {
        log(mNotifyRadioPowerChange.get(iRadioPower) + " unregisterForRadioPowerChange");
        mNotifyRadioPowerChange.remove(iRadioPower);
    }

    private static synchronized void notifyRadioPowerChange(boolean power, int phoneId) {
        for (Entry<IRadioPower, String> e : mNotifyRadioPowerChange.entrySet()) {
            log("notifyRadioPowerChange: user:" + e.getValue());
            IRadioPower iRadioPower = e.getKey();
            iRadioPower.notifyRadioPowerChange(power, phoneId);
        }
    }

    private static void log(String s) {
        Rlog.d(LOG_TAG, "[RadioManager] " + s);
    }

    public boolean isAllowAirplaneModeChange() {
        return mAirplaneRequestHandler.allowSwitching();
    }

    /**
     * Set Whether force allow airplane mode change.
     * @return true or false
     */
    public void forceAllowAirplaneModeChange(boolean forceSwitch) {
        mAirplaneRequestHandler.setForceSwitch(forceSwitch);
    }
}
