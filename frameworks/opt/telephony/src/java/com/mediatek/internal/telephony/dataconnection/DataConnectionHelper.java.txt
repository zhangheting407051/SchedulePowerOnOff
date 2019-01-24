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
 * MediaTek Inc. (C) 2016. All rights reserved.
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


package com.mediatek.internal.telephony.dataconnection;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.os.AsyncResult;
import android.os.Handler;
import android.os.Message;
import android.os.Messenger;
import android.os.SystemProperties;
import android.telephony.Rlog;
import android.telephony.ServiceState;
import android.telephony.SubscriptionManager;
import android.telephony.TelephonyManager;
import android.text.TextUtils;

import com.android.internal.telephony.Phone;
import com.android.internal.telephony.PhoneConstants;
import com.android.internal.telephony.PhoneSwitcher;
import com.android.internal.telephony.SubscriptionController;
import com.android.internal.telephony.TelephonyIntents;
import com.android.internal.telephony.uicc.IccRecords;

import com.mediatek.internal.telephony.RadioCapabilitySwitchUtil;

import java.util.List;

public class DataConnectionHelper extends Handler {
    private static final String LOG_TAG = "DcHelper";
    private static final boolean DBG = true;
    private static final boolean VDBG = SystemProperties.get("ro.build.type").
            equals("eng") ? true : false; // STOPSHIP if true

    private static DataConnectionHelper sDataConnectionHelper;
    private PhoneSwitcher mPhoneSwitcher;
    private Phone[] mPhones;
    private int mPhoneNum;
    private Context mContext;

    private static final int EVENT_VOICE_CALL_STARTED = 100;
    private static final int EVENT_VOICE_CALL_ENDED = 200;
    private static final int EVENT_NOTIFICATION_RC_CHANGED = 300;

    private static final String OPERATOR_OP09 = "OP09";
    private static final String SEGDEFAULT = "SEGDEFAULT";
    private static final String PROP_MTK_CDMA_LTE_MODE = "ro.boot.opt_c2k_lte_mode";
    public static final boolean MTK_SVLTE_SUPPORT = (SystemProperties.getInt(
            PROP_MTK_CDMA_LTE_MODE, 0) == 1);
    private static final boolean MTK_SRLTE_SUPPORT = (SystemProperties.getInt(
            PROP_MTK_CDMA_LTE_MODE, 0) == 2);
    /// M: C2K SVLTE dynamic DSDA support
    private static final int MAX_ACTIVE_PHONES_SINGLE = 1;
    private static final int MAX_ACTIVE_PHONES_DUAL = 2;

    // Multi-PS Attach
    private static final String DATA_CONFIG = "ro.mtk_data_config";

    // M: The use of return current calling phone id.
    private int mCallingPhone = SubscriptionManager.INVALID_PHONE_INDEX;

    // M: To get ICCID info.
    private String[] PROPERTY_ICCID_SIM = {
        "ril.iccid.sim1",
        "ril.iccid.sim2",
        "ril.iccid.sim3",
        "ril.iccid.sim4",
    };

    private static final String[] PROPERTY_RIL_TEST_SIM = {
        "gsm.sim.ril.testsim",
        "gsm.sim.ril.testsim.2",
        "gsm.sim.ril.testsim.3",
        "gsm.sim.ril.testsim.4",
    };

    private static final String[]  PROPERTY_RIL_FULL_UICC_TYPE = {
        "gsm.ril.fulluicctype",
        "gsm.ril.fulluicctype.2",
        "gsm.ril.fulluicctype.3",
        "gsm.ril.fulluicctype.4",
    };

    private static final String[] PROPERTY_RIL_CT3G = {
        "gsm.ril.ct3g",
        "gsm.ril.ct3g.2",
        "gsm.ril.ct3g.3",
        "gsm.ril.ct3g.4",
    };

    private static final String INVALID_ICCID = "N/A";

    private int[][] MTK_SBP_TABLE = {
        // OP129
        {44007, 44008, 129},   // Japan
        {44050, 44056, 129},   // Japan
        {44070, 44079, 129},   // Japan
        {44088, 44089, 129},   // Japan
        {44170, 44170, 129},   // Japan
    };

    private DataConnectionHelper(Context context, Phone[] phones, PhoneSwitcher phoneSwitcher) {
        mContext = context;
        mPhoneSwitcher = phoneSwitcher;
        mPhones = phones;
        mPhoneNum = phones.length;
        registerEvents();
    }

    public void dispose() {
        logd("DataConnectionHelper.dispose");
        unregisterEvents();
    }

    public static DataConnectionHelper makeDataConnectionHelper(Context context,
        Phone[] phones, PhoneSwitcher phoneSwitcher) {

        if (context == null || phones == null || phoneSwitcher == null) {
            throw new RuntimeException("param is null");
        }

        if (sDataConnectionHelper == null) {
            logd("makeDataConnectionHelper: phones.length=" + phones.length);
            sDataConnectionHelper = new DataConnectionHelper(context, phones, phoneSwitcher);
        }

        logd("makesDataConnectionHelper: X sDataConnectionHelper =" + sDataConnectionHelper);
        return sDataConnectionHelper;
    }


    private Handler mRspHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            AsyncResult ar;
            if (msg.what >= EVENT_NOTIFICATION_RC_CHANGED) {
                logd("EVENT_PHONE" + (msg.what - EVENT_NOTIFICATION_RC_CHANGED + 1)
                        + "_EVENT_NOTIFICATION_RC_CHANGED.");
                onCheckIfRetriggerDataAllowed(msg.what - EVENT_NOTIFICATION_RC_CHANGED);
            } else if (msg.what >= EVENT_VOICE_CALL_ENDED) {
                logd("EVENT_PHONE" + (msg.what - EVENT_VOICE_CALL_ENDED + 1)
                        + "_VOICE_CALL_ENDED.");
                logd("mCallingPhone = " + mCallingPhone);
                onVoiceCallEnded();
                mCallingPhone = SubscriptionManager.INVALID_PHONE_INDEX;
            } else if (msg.what >= EVENT_VOICE_CALL_STARTED) {
                logd("EVENT_PHONE" + (msg.what - EVENT_VOICE_CALL_STARTED + 1)
                        + "_VOICE_CALL_STARTED.");
                // In terms of mCallingPhone,  0 = SIM1 while 1 = SIM2
                mCallingPhone = msg.what - EVENT_VOICE_CALL_STARTED;
                logd("mCallingPhone = " + mCallingPhone);
                onVoiceCallStarted();
            }
        }
    };

    public static DataConnectionHelper getInstance() {
        if (sDataConnectionHelper == null) {
            throw new RuntimeException("Should not be called before makesDataConnectionHelper");
        }
        return sDataConnectionHelper;
    }

    /**
    * M: Multi-PS attach support or not.
    * @return boolean true if support Multi-PS attach
    */
    public static boolean isMultiPsAttachSupport() {
        int config = SystemProperties.getInt(DATA_CONFIG, 0);
        boolean support = false;

        if (config == 1) {
            support = true;
        }

        return support;
    }

    /**
    * M: To do PS re-attach, it will do PS detach then PS attach.
    */
    public void reRegisterPsNetwork() {
        mPhoneSwitcher.reRegisterPsNetwork();
    }

    // M: PS/CS concurrent feature start
    private void registerEvents() {
        logd("registerEvents");
        // M: Register events
        for (int i = 0; i < mPhoneNum; i++) {
            // Register events for call state.
            mPhones[i].getCallTracker().registerForVoiceCallStarted (mRspHandler,
                    EVENT_VOICE_CALL_STARTED + i, null);
            mPhones[i].getCallTracker().registerForVoiceCallEnded (mRspHandler,
                    EVENT_VOICE_CALL_ENDED + i, null);

            // Register event for radio capability change
            mPhones[i].registerForRadioCapabilityChanged(mRspHandler,
                    EVENT_NOTIFICATION_RC_CHANGED + i, null);
        }

        /// M: C2K SVLTE dynamic DSDA support
        if (MTK_SVLTE_SUPPORT) {
            IntentFilter filter = new IntentFilter();
            filter.addAction(TelephonyIntents.ACTION_RADIO_TECHNOLOGY_CHANGED);
            mContext.registerReceiver(mModeStateReceiver, filter);
            logd("registered mModeStateReceiver.");
        }
    }

    private void unregisterEvents() {
        logd("unregisterEvents");
        // M: Unregister events
        for (int i = 0; i < mPhoneNum; i++) {
            // Unregister events  for voice call
            mPhones[i].getCallTracker().unregisterForVoiceCallStarted(mRspHandler);
            mPhones[i].getCallTracker().unregisterForVoiceCallEnded(mRspHandler);
            // Unregister event for radio capability change
            mPhones[i].unregisterForRadioCapabilityChanged(mRspHandler);
        }
        /// M: C2K SVLTE dynamic DSDA support
        if (MTK_SVLTE_SUPPORT) {
            mContext.unregisterReceiver(mModeStateReceiver);
            logd("unregistered mModeStateReceiver.");
        }
    }

    private void onVoiceCallStarted() {
        for (int i = 0; i < mPhoneNum; i++) {
            logd("onVoiceCallStarted: mPhone[ " + i +"]");
            mPhones[i].mDcTracker.onVoiceCallStarted();
        }
    }

    private void onVoiceCallEnded() {
        for (int i = 0; i < mPhoneNum; i++) {
            logd("onVoiceCallEnded: mPhone[ " + i +"]");
            mPhones[i].mDcTracker.onVoiceCallEnded();
        }
    }

    public boolean isDataSupportConcurrent(int phoneId) {
        if (mCallingPhone == SubscriptionManager.INVALID_PHONE_INDEX) {
            logd("isDataSupportConcurrent: invalid calling phone!");
            return false;
        }

        // PS & CS on the same phone
        if (phoneId == mCallingPhone) {
            // Use sender's phone id (e.g. DcTracker) to query its services state.
            boolean isConcurrent = mPhones[phoneId].getServiceStateTracker()
                    .isConcurrentVoiceAndDataAllowed();
            logd("isDataSupportConcurrent:(PS&CS on the same phone) isConcurrent= "
                    + isConcurrent + "phoneId= " + phoneId + " mCallingPhone = " + mCallingPhone);
            return isConcurrent;
        } else {
            // PS & CS not on the same phone
            if (MTK_SRLTE_SUPPORT) {
                //  For SRLTE, return false directly since DSDS.
                logd("isDataSupportConcurrent: support SRLTE ");
                return false;
            } else if (MTK_SVLTE_SUPPORT) {
                //  For SVLTE, need to check more conditions since DSDA.
                int phoneType = mPhones[mCallingPhone].getPhoneType();

                if (phoneType == PhoneConstants.PHONE_TYPE_CDMA) {
                    // If the calling phone is CDMA type(PS on the other phone is GSM), return true.
                    return true;
                } else {
                    // If the calling phone is GSM type, need to check the other phone's PS Rat.
                    // If the other phone's PS type is CDMA, return true, else, return false.
                    int rilRat = mPhones[phoneId].getServiceState().getRilDataRadioTechnology();
                    logd("isDataSupportConcurrent: support SVLTE RilRat = " + rilRat
                            + "calling phoneType: " + phoneType);

                    return (ServiceState.isCdma(rilRat));
                }
            } else {
                logd("isDataSupportConcurrent: not SRLTE or SVLTE ");
                return false;
            }
        }
    }

    public boolean isAllCallingStateIdle() {
        PhoneConstants.State [] state = new PhoneConstants.State[mPhoneNum];
        boolean allCallingState = false;
        for (int i = 0; i < mPhoneNum; i++) {
            state[i] = mPhones[i].getCallTracker().getState();

            if (state[i] != null && state[i] == PhoneConstants.State.IDLE) {
                allCallingState = true;
            } else {
                allCallingState = false;
                break;
            }
        }

        if (!allCallingState && VDBG) {
            // For log reduction, only log if call state not IDLE and not shown in user load.
            for (int i = 0; i < mPhoneNum; i++) {
                logd("isAllCallingStateIdle: state[" + i + "]=" + state[i] +
                        " allCallingState = " + allCallingState);
            }
        }
        return allCallingState;
    }
    // M: PS/CS concurrent feature end

    private boolean isCdmaCard(int phoneId) {
        boolean isCdmaSim = false;
        if (phoneId < 0 || phoneId >= TelephonyManager.getDefault().getPhoneCount()) {
            logd("isCdmaCard invalid phoneId:" + phoneId);
            return isCdmaSim;
        }

        String cardType = SystemProperties.get(PROPERTY_RIL_FULL_UICC_TYPE[phoneId]);
        isCdmaSim = (cardType.indexOf("CSIM") >= 0 || cardType.indexOf("RUIM") >= 0);

        if (!isCdmaSim && "SIM".equals(cardType)) {
            String uimDualMode = SystemProperties.get(PROPERTY_RIL_CT3G[phoneId]);
            if ("1".equals(uimDualMode)) {
                isCdmaSim = true;
            }
        }

        return isCdmaSim;
    }

    public boolean isSimInserted(int phoneId) {
        logd("isSimInserted:phoneId =" + phoneId);
        String iccid = SystemProperties.get(PROPERTY_ICCID_SIM[phoneId], "");
        return !TextUtils.isEmpty(iccid) && !INVALID_ICCID.equals(iccid);
    }

    public boolean isTestIccCard(int phoneId) {
        String testCard = null;

        testCard = SystemProperties.get(PROPERTY_RIL_TEST_SIM[phoneId], "");
        if (VDBG) logd("isTestIccCard: phoneId id = " + phoneId + ", iccType = " + testCard);
        return (testCard != null && testCard.equals("1"));
    }

    // M: Retrigger data allow to phone switcher
    //    In the case of default data not equal main protocol,
    //    mostly in the project which swtich data without SIM switch.
    private void onCheckIfRetriggerDataAllowed(int phoneId) {
        int defDataSubId = SubscriptionController.getInstance().getDefaultDataSubId();
        int defDataPhoneId = SubscriptionManager.getPhoneId(defDataSubId);
        int mainCapPhoneId = RadioCapabilitySwitchUtil.getMainCapabilityPhoneId();

        logd("onCheckIfRetriggerDataAllowed: defDataSubId = " + defDataSubId
                + "defDataPhoneId =" + defDataPhoneId + "mainCapPhoneId = "
                + mainCapPhoneId + "mPhoneNum =" + mPhoneNum);

        // If default data phone ID is different from main protocol phone ID then resend.
        if (defDataPhoneId < mPhoneNum &&
                defDataPhoneId >= 0 && defDataPhoneId != mainCapPhoneId) {
            logd("onCheckIfRetriggerDataAllowed: retriggerDataAllowed: mPhone[" + phoneId +"]");
            mPhoneSwitcher.resendDataAllowed(phoneId);
        } else {
            logd("onCheckIfRetriggerDataAllowed: phoneId out of boundary :" + defDataPhoneId);
            defDataPhoneId = SubscriptionManager.INVALID_PHONE_INDEX;
        }
    }

    /// M: C2K SVLTE dynamic DSDA support
    private final BroadcastReceiver mModeStateReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            final String action = intent.getAction();
            if (action == null) {
                logd("mModeStateReceiver: Action is null");
                return;
            }
            /* M:
             * ACTION_RADIO_TECHNOLOGY_CHANGED is needed for roaming/hot-plug cases,
             * like roaming from C -> G, plug out C mode sim.
             */
            if (TelephonyIntents.ACTION_RADIO_TECHNOLOGY_CHANGED.equals(action)) {
                int defDataPhoneId = SubscriptionManager.getPhoneId(
                        SubscriptionController.getInstance().getDefaultDataSubId());
                logd("mModeStateReceiver: update DSDS/DSDA mode, action=" +
                        action + " defDataPhoneId = " + defDataPhoneId);
                updateMaxActivePhoneSvlte(defDataPhoneId);
                mPhoneSwitcher.onModeChanged();
            }
        }
    };

    /// M: C2K SVLTE dynamic DSDA support, if C+LG mode, two active phones.
    public void updateMaxActivePhoneSvlte(int defDataPhoneId) {
        int cdmaSlot = SubscriptionManager.INVALID_PHONE_INDEX;
        int simCount = 0;
        int cSimCount = 0;
        int mainSlot = defDataPhoneId;
        for (int i = 0; i < mPhones.length; i++) {
            if (isSimInserted(i)) {
                simCount++;
            }
            if(isCdmaCard(i)) {
                cSimCount++;
            }
            if (mPhones[i].getPhoneType() == PhoneConstants.PHONE_TYPE_CDMA) {
               cdmaSlot = i;
            }
        }
        int oldActive = mPhoneSwitcher.getMaxActivePhonesCount();
        int newActive;
        // M: default sub is not set or is not exist in device, use 4g phoneId.
        if (!SubscriptionManager.isValidPhoneId(mainSlot)) {
            logd("updateMaxActivePhoneSvlte: default sub not in device, use 4G slot.");
            mainSlot = RadioCapabilitySwitchUtil.getMainCapabilityPhoneId();
        }
        if (!isOP09ASupport() &&
            SubscriptionManager.isValidPhoneId(cdmaSlot) &&
            SubscriptionManager.isValidPhoneId(mainSlot) &&
            simCount == MAX_ACTIVE_PHONES_DUAL &&
            cSimCount != MAX_ACTIVE_PHONES_DUAL &&
            cdmaSlot != mainSlot) {
            newActive = MAX_ACTIVE_PHONES_DUAL;
        } else {
            newActive = MAX_ACTIVE_PHONES_SINGLE;
        }
        logd("updateMaxActivePhoneSvlte: cdma slot:" + cdmaSlot + ", main slot:" + mainSlot +
                ", oldActive:" + oldActive + ", now:" + newActive + ", cSimCount:" + cSimCount);
        if (newActive != oldActive) {
            mPhoneSwitcher.setMaxActivePhones(newActive);
        }
    }

    /// M: C2K SVLTE dynamic DSDA support, if C+LG mode, two active phones.
    public void updateActivePhonesSvlte(List<Integer> phones) {
        int count = mPhoneSwitcher.getMaxActivePhonesCount();
        if (count == MAX_ACTIVE_PHONES_DUAL) {
            phones.clear();
            for (int i = 0; i < mPhones.length; i++) {
                phones.add(i);
            }
        } else if (count == MAX_ACTIVE_PHONES_SINGLE) {
            /// M: When only one C SIM inserted and it is not main capability and default data card.
            ///    It may not register 3G after send MMS. So, need to add CDMA phone in this case.
            if (!phones.isEmpty()) {
                return;
            }

            int cdmaPhone = -1;
            int simCount = 0;

            for (int i = 0; i < mPhones.length; i++) {
                if (isSimInserted(i)) {
                    simCount++;
                    if (mPhones[i].getPhoneType() == PhoneConstants.PHONE_TYPE_CDMA) {
                        cdmaPhone = i;
                    }
                }
            }

            if (simCount == 1 && cdmaPhone >= 0) {
                logd("updateActivePhonesSvlte: add CDMA phone as active phone.");
                phones.add(cdmaPhone);
            }
        }
    }

    private static void logd(String s) {
        if (DBG) {
            Rlog.d(LOG_TAG, s);
        }
    }

    public int getSbpIdFromNetworkOperator(int PhoneId) {
        return getSbpId(TelephonyManager.getDefault().getNetworkOperatorForPhone(PhoneId));
    }

    public int getSbpIdFromSimOperator(IccRecords r) {
        return getSbpId((r != null) ? r.getOperatorNumeric() : "");
    }

    private int getSbpId(String strMccMnc) {
        int sbpId = -1;
        try {
            if (!TextUtils.isEmpty(strMccMnc)) {
                int mccmnc = Integer.parseInt(strMccMnc);
                for (int[] sbpEntry : MTK_SBP_TABLE) {
                    if (mccmnc >= sbpEntry[0] && mccmnc <= sbpEntry[1]) {
                        sbpId = sbpEntry[2];
                        break;
                    }
                }
            }
        } catch (NumberFormatException e) {
            logd("getSbpId: e=" + e);
            sbpId = -1;
        }
        return sbpId;
    }

    private boolean isOP09ASupport() {
        return OPERATOR_OP09.equals(SystemProperties.get("persist.operator.optr", ""))
                && SEGDEFAULT.equals(SystemProperties.get("persist.operator.seg", ""));
    }
}
