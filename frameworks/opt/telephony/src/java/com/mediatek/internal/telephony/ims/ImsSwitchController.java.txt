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
import android.os.IBinder;
import android.os.Message;
import android.os.RemoteException;
import android.os.ServiceManager;
import android.os.SystemProperties;
import android.provider.Settings;
import android.content.Intent;
import android.content.IntentFilter;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.database.ContentObserver;
import android.net.ConnectivityManager;
import android.net.NetworkInfo;
import android.net.Uri;
import android.net.wifi.WifiManager;
import android.net.wifi.WifiManager.WifiOffListener;
import android.telephony.Rlog;
import android.telephony.SubscriptionManager;
import android.telephony.TelephonyManager;
import com.android.ims.ImsConfig;
import com.android.ims.ImsManager;
import com.android.ims.internal.IImsService;
import com.android.internal.telephony.CommandsInterface;
import com.android.internal.telephony.CommandsInterface.RadioState;
import com.android.internal.telephony.PhoneConstants;
import com.android.internal.telephony.RILConstants;
import com.android.internal.telephony.TelephonyIntents;
import com.mediatek.ims.WfcReasonInfo;
public class ImsSwitchController extends Handler  {
    static final String LOG_TAG = "ImsSwitchController";
    public static final String IMS_SERVICE = "ims";
    /// M: ALPS02373062. Update NW_TYPE_WIFI string,
    /// workaround for WIFI status error in flight mode. @{
    public static final String NW_TYPE_WIFI = "MOBILE_IMS";
    /// @}
    public static final String NW_SUB_TYPE_IMS = "IMS";
    private Context mContext;
    private CommandsInterface[] mCi;
    private int mPhoneCount;
    private static IImsService mImsService = null;
    private RadioPowerInterface mRadioPowerIf;
    private boolean mIsInVoLteCall = false;
    private ImsServiceDeathRecipient mDeathRecipient = new ImsServiceDeathRecipient();
    private boolean mNeedTurnOffWifi = false;
    private int REASON_INVALID = -1;
    private int mReason = REASON_INVALID;
    protected final Object mLock = new Object();
    /** events id definition */
    protected static final int EVENT_RADIO_NOT_AVAILABLE_PHONE1    = 1;
    protected static final int EVENT_RADIO_AVAILABLE_PHONE1        = 2;
    protected static final int EVENT_RADIO_NOT_AVAILABLE_PHONE2    = 3;
    protected static final int EVENT_RADIO_AVAILABLE_PHONE2        = 4;
    protected static final int EVENT_DC_SWITCH_STATE_CHANGE        = 5;
    protected static final int EVENT_CONNECTIVITY_CHANGE           = 6;
    protected static final int EVENT_IMS_DEREGISTER_TIMEOUT        = 7;
    static final int DISABLE_WIFI_FLIGHTMODE = 1;
    private static final String PROPERTY_VOLTE_ENALBE = "persist.mtk.volte.enable";
    private static final String PROPERTY_WFC_ENALBE = "persist.mtk.wfc.enable";
    private static final String PROPERTY_IMS_VIDEO_ENALBE = "persist.mtk.ims.video.enable";
    ImsSwitchController(Context context , int phoneCount, CommandsInterface[] ci) {
        log("Initialize ImsSwitchController");
        mContext = context;
        mCi = ci;
        mPhoneCount = phoneCount;
        // For TC1, do not use MTK IMS stack solution
        if (SystemProperties.get("persist.mtk_ims_support").equals("1") &&
            !SystemProperties.get("ro.mtk_tc1_feature").equals("1")) {
            IntentFilter intentFilter = new IntentFilter(ImsManager.ACTION_IMS_SERVICE_DOWN);
            intentFilter.addAction(ImsManager.ACTION_IMS_SERVICE_DEREGISTERED);
            intentFilter.addAction(TelephonyIntents.ACTION_SET_RADIO_CAPABILITY_DONE);
            if (SystemProperties.get("persist.mtk_wfc_support").equals("1")) {
                intentFilter.addAction(WifiManager.WIFI_STATE_CHANGED_ACTION);
                intentFilter.addAction(ConnectivityManager.CONNECTIVITY_ACTION);
            }
            mContext.registerReceiver(mIntentReceiver, intentFilter);
            mRadioPowerIf = new RadioPowerInterface();
            RadioManager.registerForRadioPowerChange(LOG_TAG, mRadioPowerIf);
            mCi[PhoneConstants.SIM_ID_1].registerForNotAvailable(this,
                EVENT_RADIO_NOT_AVAILABLE_PHONE1, null);
            mCi[PhoneConstants.SIM_ID_1].registerForAvailable(this,
                EVENT_RADIO_AVAILABLE_PHONE1, null);
            if (mPhoneCount > PhoneConstants.SIM_ID_2) {
                mCi[PhoneConstants.SIM_ID_2].registerForNotAvailable(this,
                    EVENT_RADIO_NOT_AVAILABLE_PHONE2, null);
                mCi[PhoneConstants.SIM_ID_2].registerForAvailable(this,
                    EVENT_RADIO_AVAILABLE_PHONE2, null);
            }
        }
    }
    class RadioPowerInterface implements IRadioPower {
        public void notifyRadioPowerChange(boolean power, int phoneId) {
            log("notifyRadioPowerChange, power:" + power + " phoneId:" + phoneId);
            if (RadioCapabilitySwitchUtil.getMainCapabilityPhoneId() == phoneId) {
                if (mImsService == null) {
                    checkAndBindImsService(phoneId);
                }

                if (mImsService != null) {
                    try {
                        int radioState = (power ?
                                RadioState.RADIO_ON.ordinal() : RadioState.RADIO_OFF.ordinal());
                        mImsService.updateRadioState(radioState, phoneId);
                    } catch (RemoteException e) {
                        log("RemoteException can't notify power state change");
                    }
                } else {
                    log("notifyRadioPowerChange: ImsService not ready !!!");
                }
            }
            log("radio power change processed");
        }
    }
    /**
     * Death recipient class for monitoring IMS service.
     *
     * @param phoneId  to indicate which phone.
     */
    private void checkAndBindImsService(int phoneId) {
        IBinder b = ServiceManager.getService(IMS_SERVICE);
        if (b != null) {
            try {
                b.linkToDeath(mDeathRecipient, 0);
            } catch (RemoteException e) {
            }
        }
        mImsService = IImsService.Stub.asInterface(b);
        log("checkAndBindImsService: mImsService = " + mImsService);
    }
    /**
     * Death recipient class for monitoring IMS service.
     */
    private class ImsServiceDeathRecipient implements IBinder.DeathRecipient {
        @Override
        public void binderDied() {
            mImsService = null;
        }
    }
    private void registerEvent() {
        log("registerEvent, major phoneid:" + RadioCapabilitySwitchUtil.getMainCapabilityPhoneId());
    }
    private void unregisterEvent() {
        log("unregisterEvent, major phoneid:" +
            RadioCapabilitySwitchUtil.getMainCapabilityPhoneId());
    }
    private final BroadcastReceiver mIntentReceiver = new BroadcastReceiver() {
        public void onReceive(Context context, Intent intent) {
            if (intent == null) return;
            String action = intent.getAction();
            log("mIntentReceiver Receive action " + action);
        }
    };
    private String eventIdtoString(int what) {
        String str = null;
        switch (what) {
            case EVENT_RADIO_NOT_AVAILABLE_PHONE1:
                str = "RADIO_NOT_AVAILABLE_PHONE1";
                break;
            case EVENT_RADIO_NOT_AVAILABLE_PHONE2:
                str = "RADIO_NOT_AVAILABLE_PHONE2";
                break;
            case EVENT_RADIO_AVAILABLE_PHONE1:
                str = "RADIO_AVAILABLE_PHONE1";
                break;
            case EVENT_RADIO_AVAILABLE_PHONE2:
                str = "RADIO_AVAILABLE_PHONE2";
                break;
            default:
                break;
        }
        return str;
    }
    @Override
    public void handleMessage(Message msg) {
        AsyncResult ar = (AsyncResult) msg.obj;
        log("handleMessage msg.what: " + eventIdtoString(msg.what));
        int phoneId = 0;
        switch (msg.what) {
            case EVENT_RADIO_NOT_AVAILABLE_PHONE1:
                phoneId = PhoneConstants.SIM_ID_1;
                break;
            case EVENT_RADIO_NOT_AVAILABLE_PHONE2:
                phoneId = PhoneConstants.SIM_ID_2;
                break;
            case EVENT_RADIO_AVAILABLE_PHONE1:
                phoneId = PhoneConstants.SIM_ID_1;
                break;
            case EVENT_RADIO_AVAILABLE_PHONE2:
                phoneId = PhoneConstants.SIM_ID_2;
                break;
            default:
                super.handleMessage(msg);
                break;
        }
    }
    private static void log(String s) {
        Rlog.d(LOG_TAG, s);
    }
}

