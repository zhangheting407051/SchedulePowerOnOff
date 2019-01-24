/*
* Copyright (C) 2014 MediaTek Inc.
* Modification based on code covered by the mentioned copyright
* and/or permission notice(s).
*/
/*
 * Copyright (C) 2015 The Android Open Source Project
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

package com.android.internal.telephony;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.os.AsyncResult;
import android.os.Bundle;
import android.os.Handler;
import android.os.Message;
import android.os.Registrant;
import android.os.RegistrantList;
import android.os.SystemProperties;
import android.telephony.CellLocation;
import android.telephony.DisconnectCause;
import android.telephony.PhoneNumberUtils;
import android.telephony.ServiceState;
import android.telephony.TelephonyManager;
import android.telephony.cdma.CdmaCellLocation;
import android.telephony.gsm.GsmCellLocation;
import android.text.TextUtils;
import java.util.Iterator;
import android.telephony.Rlog;
import android.util.EventLog;

import com.android.internal.telephony.cdma.CdmaCallWaitingNotification;
import com.android.internal.telephony.EventLogTags;

import java.io.FileDescriptor;
import java.io.PrintWriter;
import java.util.List;
import java.util.ArrayList;

/// M: CC: For 3G VT only @{
import android.os.RemoteException;

import com.mediatek.internal.telephony.gsm.GsmVTProvider;
import com.mediatek.internal.telephony.gsm.GsmVideoCallProviderWrapper;
import com.mediatek.internal.telephony.gsm.IGsmVideoCallProvider;
import android.telecom.VideoProfile;
/// @}

/// M: CC: Use GsmCallTrackerHelper
import com.android.internal.telephony.gsm.GsmCallTrackerHelper;
import com.android.internal.telephony.imsphone.ImsPhoneConnection;

/**
 * {@hide}
 */
public class GsmCdmaCallTracker extends CallTracker {
    private static final String LOG_TAG = "GsmCdmaCallTracker";
    private static final boolean REPEAT_POLLING = false;

    private static final boolean DBG_POLL = false;
    private static final boolean VDBG = false;

    //***** Constants

    public static final int MAX_CONNECTIONS_GSM = 19;   //7 allowed in GSM + 12 from IMS for SRVCC
    private static final int MAX_CONNECTIONS_PER_CALL_GSM = 5; //only 5 connections allowed per call

    private static final int MAX_CONNECTIONS_CDMA = 8;
    private static final int MAX_CONNECTIONS_PER_CALL_CDMA = 1; //only 1 connection allowed per call

    //***** Instance Variables
    public GsmCdmaConnection mConnections[];
    private RegistrantList mVoiceCallEndedRegistrants = new RegistrantList();
    private RegistrantList mVoiceCallStartedRegistrants = new RegistrantList();

    /// M: CC: Proprietary incoming call handling @{
    public RegistrantList mVoiceCallIncomingIndicationRegistrants = new RegistrantList();
    /// @}

    // connections dropped during last poll
    private ArrayList<GsmCdmaConnection> mDroppedDuringPoll =
            new ArrayList<GsmCdmaConnection>(MAX_CONNECTIONS_GSM);

    public GsmCdmaCall mRingingCall = new GsmCdmaCall(this);
    // A call that is ringing or (call) waiting
    public GsmCdmaCall mForegroundCall = new GsmCdmaCall(this);
    public GsmCdmaCall mBackgroundCall = new GsmCdmaCall(this);

    private GsmCdmaConnection mPendingMO;
    public boolean mHangupPendingMO;

    public GsmCdmaPhone mPhone;

    private boolean mDesiredMute = false;    // false = mute off

    public PhoneConstants.State mState = PhoneConstants.State.IDLE;

    private TelephonyEventLog mEventLog;

    // Following member variables are for CDMA only
    private RegistrantList mCallWaitingRegistrants = new RegistrantList();
    private boolean mPendingCallInEcm;
    private boolean mIsInEmergencyCall;
    private int mPendingCallClirMode;
    private boolean mIsEcmTimerCanceled;
    private int m3WayCallFlashDelay;

    /// M: CC: Vzw ECC/hVoLTE redial
    private RedialState mRedialState = RedialState.REDIAL_NONE;

    /// M: CC: For 3G VT only @{
    /* voice&video waiting */
    private boolean hasPendingReplaceRequest = false;
    /// @}

    /// M: CC: Proprietary CRSS handling @{
    boolean mHasPendingSwapRequest = false;
    WaitingForHold mWaitingForHoldRequest = new WaitingForHold();
    /// @}

    /// M: CC: Use GsmCallTrackerHelper @{
    // Declare as public for GsmCallTrackerHelper to use
    GsmCallTrackerHelper mHelper;
    /// @}

    ///M: For IMS conference SRVCC @{
    private ArrayList<Connection> mImsConfParticipants = new ArrayList<Connection>();

    // for SRVCC purpose, put conference connection Ids temporarily
    private int[] mEconfSrvccConnectionIds = null;
    /// @}

    /// M: CDMA process plus code @{
    private com.mediatek.common.telephony.cdma.ICdmaCallTrackerExt mCdmaCallTrackerExt;
    /// @}

    /**
     * Listens for Emergency Callback Mode state change intents
     */
    private BroadcastReceiver mEcmExitReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            if (intent.getAction().equals(
                    TelephonyIntents.ACTION_EMERGENCY_CALLBACK_MODE_CHANGED)) {

                boolean isInEcm = intent.getBooleanExtra(PhoneConstants.PHONE_IN_ECM_STATE, false);
                log("Received ACTION_EMERGENCY_CALLBACK_MODE_CHANGED isInEcm = " + isInEcm);

                // If we exit ECM mode, notify all connections.
                if (!isInEcm) {
                    // Although mConnections seems to be the place to look, it is not guaranteed
                    // to have all of the connections we're tracking.  THe best place to look is in
                    // the Call objects associated with the tracker.
                    List<Connection> toNotify = new ArrayList<Connection>();
                    toNotify.addAll(mRingingCall.getConnections());
                    toNotify.addAll(mForegroundCall.getConnections());
                    toNotify.addAll(mBackgroundCall.getConnections());
                    if (mPendingMO != null) {
                        toNotify.add(mPendingMO);
                    }

                    // Notify connections that ECM mode exited.
                    for (Connection connection : toNotify) {
                        if (connection != null) {
                            connection.onExitedEcmMode();
                        }
                    }
                }
            }
        }
    };

    /// M: CC: Proprietary CRSS handling @{
    class WaitingForHold {

        private boolean mWaiting = false;
        private String mDialString = null;
        private int mClirMode = 0;
        private UUSInfo mUUSInfo = null;

        WaitingForHold() {
            reset();
        }

        boolean isWaiting() {
            return mWaiting;
        }

        void set() {
            mWaiting = true;
        }

        public void set(String dialSting, int clir, UUSInfo uusinfo) {
            mWaiting = true;
            mDialString = dialSting;
            mClirMode = clir;
            mUUSInfo = uusinfo;
        }

        public void reset() {

            Rlog.i(LOG_TAG, "Reset WaitingForHoldRequest variables");

            mWaiting = false;
            mDialString = null;
            mClirMode = 0;
            mUUSInfo = null;
        }

        /**
         * Check if there is another action need to be performed after holding request is done.
         *
         * @return Return true if there exists action need to be perform, else return false.
         */
        private boolean handleOperation() {
            log("handleWaitingOperation begin");

            if (mWaiting) {
                /// M: CC: ECC Retry @{
                if (PhoneNumberUtils.isEmergencyNumber(mPhone.getSubId(), mDialString)
                /// @}
                        && !PhoneNumberUtils.isSpecialEmergencyNumber(mDialString)) {
                    int serviceCategory = PhoneNumberUtils.getServiceCategoryFromEcc(mDialString);
                    mCi.setEccServiceCategory(serviceCategory);
                    mCi.emergencyDial(mDialString, mClirMode, mUUSInfo,
                            obtainCompleteMessage(EVENT_DIAL_CALL_RESULT));
                } else {
                    mCi.dial(mDialString, mClirMode, mUUSInfo,
                            obtainCompleteMessage(EVENT_DIAL_CALL_RESULT));
                }

                /// M: CC: For 3G VT only @{
                if (SystemProperties.get("ro.mtk_vt3g324m_support").equals("1")) {
                    //MO:new VT service
                    mForegroundCall.mVTProvider = new GsmVTProvider();
                    log("handleOperation new GsmVTProvider");
                    try {
                        IGsmVideoCallProvider gsmVideoCallProvider =
                                mForegroundCall.mVTProvider.getInterface();
                        if (gsmVideoCallProvider != null) {
                            GsmVideoCallProviderWrapper gsmVideoCallProviderWrapper =
                                    new GsmVideoCallProviderWrapper(gsmVideoCallProvider);
                            log("handleOperation new GsmVideoCallProviderWrapper");
                            mPendingMO.setVideoProvider(gsmVideoCallProviderWrapper);
                        }
                    } catch (RemoteException e) {
                        Rlog.e(LOG_TAG, "handleOperation new GsmVideoCallProviderWrapper failed");
                    }
                }
                /// @}

                reset();
                log("handleWaitingOperation end");
                return true;
            }
            return false;
        }
    }
    /// @}

    //***** Events


    //***** Constructors

    public GsmCdmaCallTracker (GsmCdmaPhone phone) {
        this.mPhone = phone;
        mCi = phone.mCi;
        mCi.registerForCallStateChanged(this, EVENT_CALL_STATE_CHANGE, null);
        mCi.registerForOn(this, EVENT_RADIO_AVAILABLE, null);
        mCi.registerForNotAvailable(this, EVENT_RADIO_NOT_AVAILABLE, null);

        /// M: CC: Proprietary incoming call handling
        mCi.setOnIncomingCallIndication(this, EVENT_INCOMING_CALL_INDICATION, null);
        /// M: CC: Modem reset related handling
        mCi.registerForOffOrNotAvailable(this, EVENT_RADIO_OFF_OR_NOT_AVAILABLE, null);
        /// M: CC: Vzw ECC/hVoLTE redial
        mCi.registerForCallRedialState(this, EVENT_CALL_REDIAL_STATE, null);


        // Register receiver for ECM exit
        IntentFilter filter = new IntentFilter();
        filter.addAction(TelephonyIntents.ACTION_EMERGENCY_CALLBACK_MODE_CHANGED);
        mPhone.getContext().registerReceiver(mEcmExitReceiver, filter);

        updatePhoneType(true);

        mEventLog = new TelephonyEventLog(mPhone.getPhoneId());

        /// M: CC: Use GsmCallTrackerHelper @{
        mHelper = new GsmCallTrackerHelper(phone.getContext(), this);
        /// @}
        ///M: For IMS conference SRVCC @{
        mCi.registerForEconfSrvcc(this, EVENT_ECONF_SRVCC_INDICATION, null);
        /// @}
        /// M: CDMA process plus code @{
        mCdmaCallTrackerExt = com.mediatek.common.MPlugin.createInstance(
                com.mediatek.common.telephony.cdma.ICdmaCallTrackerExt.class.getName());
        log("mCdmaCallTrackerExt:" + mCdmaCallTrackerExt);
        /// @}
    }

    public void updatePhoneType() {
        updatePhoneType(false);
    }

    private void updatePhoneType(boolean duringInit) {
        if (!duringInit) {
            reset();
            pollCallsWhenSafe();
        }

        if (mPhone.isPhoneTypeGsm()) {
            mConnections = new GsmCdmaConnection[MAX_CONNECTIONS_GSM];
            mCi.unregisterForCallWaitingInfo(this);
            /// M: CDMA process call accepted @{
            mCi.unregisterForCallAccepted(this);
            /// @}
        } else {
            mConnections = new GsmCdmaConnection[MAX_CONNECTIONS_CDMA];
            mPendingCallInEcm = false;
            mIsInEmergencyCall = false;
            mPendingCallClirMode = CommandsInterface.CLIR_DEFAULT;
            mIsEcmTimerCanceled = false;
            m3WayCallFlashDelay = 0;
            mCi.registerForCallWaitingInfo(this, EVENT_CALL_WAITING_INFO_CDMA, null);
            /// M: CDMA process call accepted @{
            mCi.registerForCallAccepted(this, EVENT_CDMA_CALL_ACCEPTED, null);
            /// @}
        }
    }

    private void reset() {
        Rlog.d(LOG_TAG, "reset");

        /// M: CC: Vzw ECC/hVoLTE redial @{
        if (mRedialState == RedialState.REDIAL_NONE) {
            handlePollCalls(new AsyncResult(null, null,
                    new CommandException(CommandException.Error.RADIO_NOT_AVAILABLE)));
        }
        /// @}

        clearDisconnected();

        for (GsmCdmaConnection gsmCdmaConnection : mConnections) {
            if (gsmCdmaConnection != null) {
                gsmCdmaConnection.dispose();
            }
        }
    }

    @Override
    protected void finalize() {
        Rlog.d(LOG_TAG, "GsmCdmaCallTracker finalized");
    }

    //***** Instance Methods

    //***** Public Methods
    @Override
    public void registerForVoiceCallStarted(Handler h, int what, Object obj) {
        Registrant r = new Registrant(h, what, obj);
        mVoiceCallStartedRegistrants.add(r);
        // Notify if in call when registering
        if (mState != PhoneConstants.State.IDLE) {
            r.notifyRegistrant(new AsyncResult(null, null, null));
        }
    }

    @Override
    public void unregisterForVoiceCallStarted(Handler h) {
        mVoiceCallStartedRegistrants.remove(h);
    }

    @Override
    public void registerForVoiceCallEnded(Handler h, int what, Object obj) {
        Registrant r = new Registrant(h, what, obj);
        mVoiceCallEndedRegistrants.add(r);
    }

    @Override
    public void unregisterForVoiceCallEnded(Handler h) {
        mVoiceCallEndedRegistrants.remove(h);
    }

    public void registerForCallWaiting(Handler h, int what, Object obj) {
        Registrant r = new Registrant (h, what, obj);
        mCallWaitingRegistrants.add(r);
    }

    public void unregisterForCallWaiting(Handler h) {
        mCallWaitingRegistrants.remove(h);
    }

    private void fakeHoldForegroundBeforeDial() {
        List<Connection> connCopy;

        // We need to make a copy here, since fakeHoldBeforeDial()
        // modifies the lists, and we don't want to reverse the order
        connCopy = (List<Connection>) mForegroundCall.mConnections.clone();

        for (int i = 0, s = connCopy.size() ; i < s ; i++) {
            GsmCdmaConnection conn = (GsmCdmaConnection)connCopy.get(i);

            conn.fakeHoldBeforeDial();
        }
    }

    //GSM
    /**
     * clirMode is one of the CLIR_ constants
     */
    public synchronized Connection dial(String dialString, int clirMode, UUSInfo uusInfo,
                                        Bundle intentExtras)
            throws CallStateException {
        // note that this triggers call state changed notif
        clearDisconnected();

        if (!canDial()) {
            throw new CallStateException("cannot dial in current state");
        }

        String origNumber = dialString;
        dialString = convertNumberIfNecessary(mPhone, dialString);

        // The new call must be assigned to the foreground call.
        // That call must be idle, so place anything that's
        // there on hold
        if (mForegroundCall.getState() == GsmCdmaCall.State.ACTIVE) {
            // this will probably be done by the radio anyway
            // but the dial might fail before this happens
            // and we need to make sure the foreground call is clear
            // for the newly dialed connection

            /// M: CC: Proprietary CRSS handling @{
            mWaitingForHoldRequest.set();
            /// @}

            switchWaitingOrHoldingAndActive();
            // This is a hack to delay DIAL so that it is sent out to RIL only after
            // EVENT_SWITCH_RESULT is received. We've seen failures when adding a new call to
            // multi-way conference calls due to DIAL being sent out before SWITCH is processed
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                // do nothing
            }

            // Fake local state so that
            // a) foregroundCall is empty for the newly dialed connection
            // b) hasNonHangupStateChanged remains false in the
            // next poll, so that we don't clear a failed dialing call
            fakeHoldForegroundBeforeDial();
        }

        if (mForegroundCall.getState() != GsmCdmaCall.State.IDLE) {
            //we should have failed in !canDial() above before we get here
            throw new CallStateException("cannot dial in current state");
        }

        mPendingMO = new GsmCdmaConnection(mPhone, checkForTestEmergencyNumber(dialString),
                this, mForegroundCall);
        mHangupPendingMO = false;

        if ( mPendingMO.getAddress() == null || mPendingMO.getAddress().length() == 0
                || mPendingMO.getAddress().indexOf(PhoneNumberUtils.WILD) >= 0) {
            // Phone number is invalid
            mPendingMO.mCause = DisconnectCause.INVALID_NUMBER;

            /// M: CC: Proprietary CRSS handling @{
            mWaitingForHoldRequest.reset();
            /// @}

            // handlePollCalls() will notice this call not present
            // and will mark it as dropped.
            pollCallsWhenSafe();
        } else {
            // Always unmute when initiating a new call
            setMute(false);
            /// M: CC: Proprietary CRSS handling @{
            //mCi.dial(mPendingMO.getAddress(), clirMode, uusInfo, obtainCompleteMessage());
            if (!mWaitingForHoldRequest.isWaiting()) {
                /// M: CC: Proprietary ECC handling@{
                /// M: CC: ECC Retry @{
                if (PhoneNumberUtils.isEmergencyNumber(mPhone.getSubId(), dialString)
                /// @}
                        && !PhoneNumberUtils.isSpecialEmergencyNumber(dialString)) {
                    int serviceCategory = PhoneNumberUtils.getServiceCategoryFromEcc(dialString);
                    mCi.setEccServiceCategory(serviceCategory);
                    mCi.emergencyDial(mPendingMO.getAddress(), clirMode, uusInfo,
                            obtainCompleteMessage(EVENT_DIAL_CALL_RESULT));
                /// @}
                } else {
                    mCi.dial(mPendingMO.getAddress(), clirMode, uusInfo,
                            obtainCompleteMessage(EVENT_DIAL_CALL_RESULT));
                }
            } else {
                mWaitingForHoldRequest.set(mPendingMO.getAddress(), clirMode, uusInfo);
            }
            /// @}

        }

        if (mNumberConverted) {
            mPendingMO.setConverted(origNumber);
            mNumberConverted = false;
        }

        updatePhoneState();
        mPhone.notifyPreciseCallStateChanged();

        return mPendingMO;
    }

    //CDMA
    /**
     * Handle Ecm timer to be canceled or re-started
     */
    private void handleEcmTimer(int action) {
        mPhone.handleTimerInEmergencyCallbackMode(action);
        switch(action) {
            case GsmCdmaPhone.CANCEL_ECM_TIMER: mIsEcmTimerCanceled = true; break;
            case GsmCdmaPhone.RESTART_ECM_TIMER: mIsEcmTimerCanceled = false; break;
            default:
                Rlog.e(LOG_TAG, "handleEcmTimer, unsupported action " + action);
        }
    }

    //CDMA
    /**
     * Disable data call when emergency call is connected
     */
    private void disableDataCallInEmergencyCall(String dialString) {
        /// M: ECC Retry @{
        //if (PhoneNumberUtils.isLocalEmergencyNumber(mPhone.getContext(), dialString)) {
        if (PhoneNumberUtils.isLocalEmergencyNumber(
                mPhone.getContext(), mPhone.getSubId(), dialString)) {
        /// @}
            if (Phone.DEBUG_PHONE) log("disableDataCallInEmergencyCall");
            setIsInEmergencyCall();
        }
    }

    //CDMA
    public void setIsInEmergencyCall() {
        mIsInEmergencyCall = true;
        mPhone.mDcTracker.setInternalDataEnabled(false);
        mPhone.notifyEmergencyCallRegistrants(true);
        mPhone.sendEmergencyCallStateChange(true);
    }

    //CDMA
    /**
     * clirMode is one of the CLIR_ constants
     */
    private Connection dial(String dialString, int clirMode) throws CallStateException {
        // note that this triggers call state changed notif
        clearDisconnected();

        if (!canDial()) {
            throw new CallStateException("cannot dial in current state");
        }

        TelephonyManager tm =
                (TelephonyManager) mPhone.getContext().getSystemService(Context.TELEPHONY_SERVICE);
        String origNumber = dialString;
        String operatorIsoContry = tm.getNetworkCountryIsoForPhone(mPhone.getPhoneId());
        String simIsoContry = tm.getSimCountryIsoForPhone(mPhone.getPhoneId());
        boolean internationalRoaming = !TextUtils.isEmpty(operatorIsoContry)
                && !TextUtils.isEmpty(simIsoContry)
                && !simIsoContry.equals(operatorIsoContry);
        if (internationalRoaming) {
            if ("us".equals(simIsoContry)) {
                internationalRoaming = internationalRoaming && !"vi".equals(operatorIsoContry);
            } else if ("vi".equals(simIsoContry)) {
                internationalRoaming = internationalRoaming && !"us".equals(operatorIsoContry);
            }
        }
        if (internationalRoaming) {
            dialString = convertNumberIfNecessary(mPhone, dialString);
        }

        /// M: Get the property by phoneId @{
        //String inEcm=SystemProperties.get(TelephonyProperties.PROPERTY_INECM_MODE, "false");
        String inEcm = mPhone.getSystemProperty(TelephonyProperties.PROPERTY_INECM_MODE, "false");
        /// @}
        boolean isPhoneInEcmMode = inEcm.equals("true");
        boolean isEmergencyCall =
                /// M: ECC Retry @{
                //PhoneNumberUtils.isLocalEmergencyNumber(mPhone.getContext(), dialString);
                PhoneNumberUtils.isLocalEmergencyNumber(
                        mPhone.getContext(), mPhone.getSubId(), dialString);
                /// @}

        // Cancel Ecm timer if a second emergency call is originating in Ecm mode
        if (isPhoneInEcmMode && isEmergencyCall) {
            handleEcmTimer(GsmCdmaPhone.CANCEL_ECM_TIMER);
        }

        // The new call must be assigned to the foreground call.
        // That call must be idle, so place anything that's
        // there on hold
        if (mForegroundCall.getState() == GsmCdmaCall.State.ACTIVE) {
            return dialThreeWay(dialString);
        }

        mPendingMO = new GsmCdmaConnection(mPhone, checkForTestEmergencyNumber(dialString),
                this, mForegroundCall);
        mHangupPendingMO = false;

        /// M: Timing issue, mPendingMO will be reset to null in handlePollCalls @{
        GsmCdmaConnection result = mPendingMO;
        /// @}

        if ( mPendingMO.getAddress() == null || mPendingMO.getAddress().length() == 0
                || mPendingMO.getAddress().indexOf(PhoneNumberUtils.WILD) >= 0 ) {
            // Phone number is invalid
            mPendingMO.mCause = DisconnectCause.INVALID_NUMBER;

            // handlePollCalls() will notice this call not present
            // and will mark it as dropped.
            pollCallsWhenSafe();
        } else {
            // Always unmute when initiating a new call
            setMute(false);

            // Check data call
            disableDataCallInEmergencyCall(dialString);

            // In Ecm mode, if another emergency call is dialed, Ecm mode will not exit.
            if(!isPhoneInEcmMode || (isPhoneInEcmMode && isEmergencyCall)) {
                /// M: CDMA FDN @{
                String tmpStr = mPendingMO.getAddress() + ","
                        + PhoneNumberUtils.extractNetworkPortionAlt(dialString);
                /// @}
                /// M: Proprietary ECC handling @{
                //mCi.dial(mPendingMO.getAddress(), clirMode, obtainCompleteMessage());
                if (isEmergencyCall) {
                    mCi.emergencyDial(tmpStr, clirMode, null,
                            obtainCompleteMessage());
                } else {
                    mCi.dial(tmpStr, clirMode, obtainCompleteMessage());
                }
                /// @}
                /// M: CDMA process plus code @{
                if (mCdmaCallTrackerExt != null && mCdmaCallTrackerExt.needToConvert(dialString,
                        GsmCdmaConnection.formatDialString(dialString))) {
                    result.setConverted(PhoneNumberUtils.extractNetworkPortionAlt(dialString));
                }
                /// @}
            } else {
                mPhone.exitEmergencyCallbackMode();
                /// M: CDMA process plus code @{
                //mPhone.setOnEcbModeExitResponse(this,EVENT_EXIT_ECM_RESPONSE_CDMA, null);
                mPhone.setOnEcbModeExitResponse(this, EVENT_EXIT_ECM_RESPONSE_CDMA, dialString);
                /// @}
                mPendingCallClirMode=clirMode;
                mPendingCallInEcm=true;
            }
        }

        if (mNumberConverted) {
            /// M: Timing issue, mPendingMO will be reset to null in handlePollCalls @{
            //mPendingMO.setConverted(origNumber);
            result.setConverted(origNumber);
            /// @}
            mNumberConverted = false;
        }

        updatePhoneState();
        mPhone.notifyPreciseCallStateChanged();

        /// M: Timing issue, mPendingMO will be reset to null in handlePollCalls @{
        //return mPendingMO;
        return result;
        /// @}
    }

    //CDMA
    private Connection dialThreeWay(String dialString) {
        if (!mForegroundCall.isIdle()) {
            // Check data call
            disableDataCallInEmergencyCall(dialString);

            // Attach the new connection to foregroundCall
            mPendingMO = new GsmCdmaConnection(mPhone,
                    checkForTestEmergencyNumber(dialString), this, mForegroundCall);
            // Some network need a empty flash before sending the normal one
            m3WayCallFlashDelay = mPhone.getContext().getResources()
                    .getInteger(com.android.internal.R.integer.config_cdma_3waycall_flash_delay);
            if (m3WayCallFlashDelay > 0) {
                /// M: CDMA process plus code @{
                //mCi.sendCDMAFeatureCode("", obtainMessage(EVENT_THREE_WAY_DIAL_BLANK_FLASH));
                mCi.sendCDMAFeatureCode("", obtainMessage(EVENT_THREE_WAY_DIAL_BLANK_FLASH,
                        dialString));
                /// @}
            } else {
                /// M: CDMA FDN @{
                String tmpStr = mPendingMO.getAddress() + ","
                        + PhoneNumberUtils.extractNetworkPortionAlt(dialString);
                /// @}
                mCi.sendCDMAFeatureCode(tmpStr,
                        obtainMessage(EVENT_THREE_WAY_DIAL_L2_RESULT_CDMA));
                /// M: CDMA process plus code @{
                if (mCdmaCallTrackerExt != null && mCdmaCallTrackerExt.needToConvert(dialString,
                        GsmCdmaConnection.formatDialString(dialString))) {
                    mPendingMO.setConverted(PhoneNumberUtils.extractNetworkPortionAlt(dialString));
                }
                /// @}
            }
            return mPendingMO;
        }
        return null;
    }

    public Connection dial(String dialString) throws CallStateException {
        if (isPhoneTypeGsm()) {
            return dial(dialString, CommandsInterface.CLIR_DEFAULT, null);
        } else {
            return dial(dialString, CommandsInterface.CLIR_DEFAULT);
        }
    }

    //GSM
    public Connection dial(String dialString, UUSInfo uusInfo, Bundle intentExtras)
            throws CallStateException {
        return dial(dialString, CommandsInterface.CLIR_DEFAULT, uusInfo, intentExtras);
    }

    //GSM
    private Connection dial(String dialString, int clirMode, Bundle intentExtras)
            throws CallStateException {
        return dial(dialString, clirMode, null, intentExtras);
    }

    public void acceptCall() throws CallStateException {
        // FIXME if SWITCH fails, should retry with ANSWER
        // in case the active/holding call disappeared and this
        // is no longer call waiting

        if (mRingingCall.getState() == GsmCdmaCall.State.INCOMING) {
            Rlog.i("phone", "acceptCall: incoming...");
            // Always unmute when answering a new call
            setMute(false);
            mCi.acceptCall(obtainCompleteMessage());
        } else if (mRingingCall.getState() == GsmCdmaCall.State.WAITING) {
            if (isPhoneTypeGsm()) {
                setMute(false);
            } else {
                GsmCdmaConnection cwConn = (GsmCdmaConnection)(mRingingCall.getLatestConnection());

                // Since there is no network response for supplimentary
                // service for CDMA, we assume call waiting is answered.
                // ringing Call state change to idle is in GsmCdmaCall.detach
                // triggered by updateParent.
                cwConn.updateParent(mRingingCall, mForegroundCall);
                cwConn.onConnectedInOrOut();
                updatePhoneState();
            }
            switchWaitingOrHoldingAndActive();
        } else {
            throw new CallStateException("phone not ringing");
        }
    }

    public void rejectCall() throws CallStateException {
        // AT+CHLD=0 means "release held or UDUB"
        // so if the phone isn't ringing, this could hang up held
        if (mRingingCall.getState().isRinging()) {
            mCi.rejectCall(obtainCompleteMessage());
        } else {
            throw new CallStateException("phone not ringing");
        }
    }

    //CDMA
    private void flashAndSetGenericTrue() {
        mCi.sendCDMAFeatureCode("", obtainMessage(EVENT_SWITCH_RESULT));

        mPhone.notifyPreciseCallStateChanged();
    }

    public void switchWaitingOrHoldingAndActive() throws CallStateException {
        // Should we bother with this check?
        if (mRingingCall.getState() == GsmCdmaCall.State.INCOMING) {
            throw new CallStateException("cannot be in the incoming state");
        } else {
            if (isPhoneTypeGsm()) {
                /// M: CC: Proprietary CRSS handling @{
                //mCi.switchWaitingOrHoldingAndActive(
                //        obtainCompleteMessage(EVENT_SWITCH_RESULT));
                if (!mHasPendingSwapRequest) {
                    mCi.switchWaitingOrHoldingAndActive(
                            obtainCompleteMessage(EVENT_SWITCH_RESULT));
                    mHasPendingSwapRequest = true;
                }
            /// @}
            } else {
                if (mForegroundCall.getConnections().size() > 1) {
                    flashAndSetGenericTrue();
                } else {
                    // Send a flash command to CDMA network for putting the other party on hold.
                    // For CDMA networks which do not support this the user would just hear a beep
                    // from the network. For CDMA networks which do support it will put the other
                    // party on hold.
                    mCi.sendCDMAFeatureCode("", obtainMessage(EVENT_SWITCH_RESULT));
                }
            }
        }
    }

    public void conference() {
        if (isPhoneTypeGsm()) {
            mCi.conference(obtainCompleteMessage(EVENT_CONFERENCE_RESULT));
        } else {
            // Should we be checking state?
            flashAndSetGenericTrue();
        }
    }

    public void explicitCallTransfer() {
        mCi.explicitCallTransfer(obtainCompleteMessage(EVENT_ECT_RESULT));
    }

    public void clearDisconnected() {
        internalClearDisconnected();

        updatePhoneState();
        mPhone.notifyPreciseCallStateChanged();
    }

    public boolean canConference() {
        return mForegroundCall.getState() == GsmCdmaCall.State.ACTIVE
                && mBackgroundCall.getState() == GsmCdmaCall.State.HOLDING
                && !mBackgroundCall.isFull()
                && !mForegroundCall.isFull();
    }

    private boolean canDial() {
        boolean ret;
        int serviceState = mPhone.getServiceState().getState();
        String disableCall = SystemProperties.get(
                TelephonyProperties.PROPERTY_DISABLE_CALL, "false");

        ret = (serviceState != ServiceState.STATE_POWER_OFF)
                && mPendingMO == null
                && !mRingingCall.isRinging()
                && !disableCall.equals("true")
                && (!mForegroundCall.getState().isAlive()
                    || !mBackgroundCall.getState().isAlive()
                    || (!isPhoneTypeGsm()
                        && mForegroundCall.getState() == GsmCdmaCall.State.ACTIVE));

        if (!ret) {
            log(String.format("canDial is false\n" +
                            "((serviceState=%d) != ServiceState.STATE_POWER_OFF)::=%s\n" +
                            "&& pendingMO == null::=%s\n" +
                            "&& !ringingCall.isRinging()::=%s\n" +
                            "&& !disableCall.equals(\"true\")::=%s\n" +
                            "&& (!foregroundCall.getState().isAlive()::=%s\n" +
                            "   || foregroundCall.getState() == GsmCdmaCall.State.ACTIVE::=%s\n" +
                            "   ||!backgroundCall.getState().isAlive())::=%s)",
                    serviceState,
                    serviceState != ServiceState.STATE_POWER_OFF,
                    mPendingMO == null,
                    !mRingingCall.isRinging(),
                    !disableCall.equals("true"),
                    !mForegroundCall.getState().isAlive(),
                    mForegroundCall.getState() == GsmCdmaCall.State.ACTIVE,
                    !mBackgroundCall.getState().isAlive()));
        }

        return ret;
    }
    public boolean canTransfer() {
        if (isPhoneTypeGsm()) {
            return (mForegroundCall.getState() == GsmCdmaCall.State.ACTIVE
                    || mForegroundCall.getState() == GsmCdmaCall.State.ALERTING
                    || mForegroundCall.getState() == GsmCdmaCall.State.DIALING)
                    && mBackgroundCall.getState() == GsmCdmaCall.State.HOLDING;
        } else {
            Rlog.e(LOG_TAG, "canTransfer: not possible in CDMA");
            return false;
        }
    }

    //***** Private Instance Methods

    private void internalClearDisconnected() {
        mRingingCall.clearDisconnected();
        mForegroundCall.clearDisconnected();
        mBackgroundCall.clearDisconnected();
    }

    /**
     * Obtain a message to use for signalling "invoke getCurrentCalls() when
     * this operation and all other pending operations are complete
     */
    /// M: CC: Use GsmCallTrackerHelper @{
    // Declare as public (not priviate) for GsmCallTrackerHelper to use
    public Message
    /// @}
    obtainCompleteMessage() {
        return obtainCompleteMessage(EVENT_OPERATION_COMPLETE);
    }

    /**
     * Obtain a message to use for signalling "invoke getCurrentCalls() when
     * this operation and all other pending operations are complete
     */
    /// M: CC: Use GsmCallTrackerHelper @{
    // Declare as public (not priviate) for GsmCallTrackerHelper to use
    public Message
    /// @}
    obtainCompleteMessage(int what) {
        mPendingOperations++;
        mLastRelevantPoll = null;
        mNeedsPoll = true;

        if (DBG_POLL) log("obtainCompleteMessage: pendingOperations=" +
                mPendingOperations + ", needsPoll=" + mNeedsPoll);

        return obtainMessage(what);
    }

    private void operationComplete() {
        mPendingOperations--;

        if (DBG_POLL) log("operationComplete: pendingOperations=" +
                mPendingOperations + ", needsPoll=" + mNeedsPoll);

        if (mPendingOperations == 0 && mNeedsPoll) {
            mLastRelevantPoll = obtainMessage(EVENT_POLL_CALLS_RESULT);
            mCi.getCurrentCalls(mLastRelevantPoll);
        } else if (mPendingOperations < 0) {
            // this should never happen
            Rlog.e(LOG_TAG,"GsmCdmaCallTracker.pendingOperations < 0");
            mPendingOperations = 0;
        }
    }

    private void updatePhoneState() {
        PhoneConstants.State oldState = mState;
        if (mRingingCall.isRinging()) {
            mState = PhoneConstants.State.RINGING;
        } else if (mPendingMO != null ||
                !(mForegroundCall.isIdle() && mBackgroundCall.isIdle())) {
            mState = PhoneConstants.State.OFFHOOK;
        } else {
            Phone imsPhone = mPhone.getImsPhone();
            /// M: ALPS02192901. @{
            // If the call is disconnected after CIREPH=1, before +CLCC, the original state is
            // idle and new state is still idle, so callEndCleanupHandOverCallIfAny isn't called.
            // Related CR: ALPS02015368, ALPS02161020, ALPS02192901.
            // if ( mState == PhoneConstants.State.OFFHOOK && (imsPhone != null)){
            if (imsPhone != null) {
                /// @}
                imsPhone.callEndCleanupHandOverCallIfAny();
            }
            mState = PhoneConstants.State.IDLE;
        }

        if (mState == PhoneConstants.State.IDLE && oldState != mState) {
            mVoiceCallEndedRegistrants.notifyRegistrants(
                new AsyncResult(null, null, null));
        } else if (oldState == PhoneConstants.State.IDLE && oldState != mState) {
            mVoiceCallStartedRegistrants.notifyRegistrants (
                    new AsyncResult(null, null, null));
        }
        if (Phone.DEBUG_PHONE) {
            log("update phone state, old=" + oldState + " new="+ mState);
        }
        if (mState != oldState) {
            mPhone.notifyPhoneStateChanged();
            mEventLog.writePhoneState(mState);
        }
    }

    // ***** Overwritten from CallTracker

    @Override
    protected synchronized void handlePollCalls(AsyncResult ar) {
        List polledCalls;

        if (VDBG) log("handlePollCalls");
        if (ar.exception == null) {
            polledCalls = (List)ar.result;
        } else if (isCommandExceptionRadioNotAvailable(ar.exception)) {
            // just a dummy empty ArrayList to cause the loop
            // to hang up all the calls
            polledCalls = new ArrayList();
        } else if (mNeedWaitImsEConfSrvcc) {
            /// M: For IMS conference SRVCC @{
            // ALPS02019630. Needs to wait +ECONFSRVCC then the call number could be known.
            log("SRVCC: +ECONFSRVCC is still not arrival, skip this poll call.");
            return;
            /// @}
        } else {
            // Radio probably wasn't ready--try again in a bit
            // But don't keep polling if the channel is closed
            pollCallsAfterDelay();
            return;
        }

        Connection newRinging = null; //or waiting
        ArrayList<Connection> newUnknownConnectionsGsm = new ArrayList<Connection>();
        Connection newUnknownConnectionCdma = null;
        boolean hasNonHangupStateChanged = false;   // Any change besides
                                                    // a dropped connection
        boolean hasAnyCallDisconnected = false;
        boolean needsPollDelay = false;
        boolean unknownConnectionAppeared = false;

        //CDMA
        boolean noConnectionExists = true;

        for (int i = 0, curDC = 0, dcSize = polledCalls.size()
                ; i < mConnections.length; i++) {
            GsmCdmaConnection conn = mConnections[i];
            DriverCall dc = null;

            // polledCall list is sparse
            if (curDC < dcSize) {
                dc = (DriverCall) polledCalls.get(curDC);

                /// M: CDMA process plus code @{
                if (!isPhoneTypeGsm() && mCdmaCallTrackerExt != null) {
                    dc.number = mCdmaCallTrackerExt.processPlusCodeForDriverCall(
                            dc.number, dc.isMT, dc.TOA);
                }
                /// @}

                if (dc.index == i+1) {
                    curDC++;
                } else {
                    dc = null;
                }
            }

            //CDMA
            if (conn != null || dc != null) {
                noConnectionExists = false;
            }

            if (DBG_POLL) log("poll: conn[i=" + i + "]=" +
                    conn+", dc=" + dc);

            if (conn == null && dc != null) {

                /// M: CC
                if (DBG_POLL) log("case 1 : new Call appear");

                // Connection appeared in CLCC response that we don't know about
                if (mPendingMO != null && mPendingMO.compareTo(dc)) {

                    if (DBG_POLL) log("poll: pendingMO=" + mPendingMO);

                    /// M: CC: For 3G VT only @{
                    //MO:set id to VT service
                    if (SystemProperties.get("ro.mtk_vt3g324m_support").equals("1")) {
                        if ((mForegroundCall.mVTProvider != null) && dc.isVideo) {
                            mForegroundCall.mVTProvider.setId(i + 1);
                        }
                    }
                    /// @}

                    // It's our pending mobile originating call
                    mConnections[i] = mPendingMO;
                    mPendingMO.mIndex = i;
                    mPendingMO.update(dc);
                    mPendingMO = null;

                    // Someone has already asked to hangup this call
                    if (mHangupPendingMO) {
                        mHangupPendingMO = false;

                        // Re-start Ecm timer when an uncompleted emergency call ends
                        if (!isPhoneTypeGsm() && mIsEcmTimerCanceled) {
                            handleEcmTimer(GsmCdmaPhone.RESTART_ECM_TIMER);
                        }

                        try {
                            if (Phone.DEBUG_PHONE) log(
                                    "poll: hangupPendingMO, hangup conn " + i);
                            hangup(mConnections[i]);
                        } catch (CallStateException ex) {
                            Rlog.e(LOG_TAG, "unexpected error on hangup");
                        }

                        // Do not continue processing this poll
                        // Wait for hangup and repoll
                        return;
                    }
                } else {
                    if (Phone.DEBUG_PHONE) {
                        log("pendingMo=" + mPendingMO + ", dc=" + dc);
                    }

                    /// M: CC: Remove handling for MO/MT conflict, not hangup MT @{
                    if (mPendingMO != null && !mPendingMO.compareTo(dc)) {
                        log("MO/MT conflict! MO should be hangup by MD");
                    }
                    /// @}

                    mConnections[i] = new GsmCdmaConnection(mPhone, dc, this, i);

                    /// M: CC: Forwarding number via EAIC @{
                    if (isPhoneTypeGsm()) {
                        //To store forwarding address to connection object.
                        mHelper.setForwardingAddressToConnection(i, mConnections[i]);
                    }
                    /// @}

                    Connection hoConnection = getHoConnection(dc);
                    if (hoConnection != null) {
                        // Single Radio Voice Call Continuity (SRVCC) completed
                        /// M: modified to fulfill IMS conference SRVCC. @{
                        if (hoConnection.isMultipartyBeforeHandover()
                                && hoConnection.isConfHostBeforeHandover()) {
                            Rlog.i(LOG_TAG, "SRVCC: goes to conference case.");
                            mConnections[i].mOrigConnection = hoConnection;
                            mImsConfParticipants.add(mConnections[i]);
                        } else {
                            Rlog.i(LOG_TAG, "SRVCC: goes to normal call case.");
                            /// @}
                            mConnections[i].migrateFrom(hoConnection);
                            // Updating connect time for silent redial cases (ex: Calls are transferred
                            // from DIALING/ALERTING/INCOMING/WAITING to ACTIVE)
                            if (hoConnection.mPreHandoverState != GsmCdmaCall.State.ACTIVE &&
                                hoConnection.mPreHandoverState != GsmCdmaCall.State.HOLDING &&
                                dc.state == DriverCall.State.ACTIVE) {
                            mConnections[i].onConnectedInOrOut();
                        }

                            mHandoverConnections.remove(hoConnection);

                            if (isPhoneTypeGsm()) {
                                for (Iterator<Connection> it = mHandoverConnections.iterator();
                                     it.hasNext(); ) {
                                    Connection c = it.next();
                                    Rlog.i(LOG_TAG, "HO Conn state is " + c.mPreHandoverState);
                                    if (c.mPreHandoverState == mConnections[i].getState()) {
                                        Rlog.i(LOG_TAG, "Removing HO conn "
                                                + hoConnection + c.mPreHandoverState);
                                        it.remove();
                                    }
                                }
                            }

                            mPhone.notifyHandoverStateChanged(mConnections[i]);
                        }
                    } else {
                        // find if the MT call is a new ring or unknown connection
                        newRinging = checkMtFindNewRinging(dc,i);
                        if (newRinging == null) {
                            unknownConnectionAppeared = true;
                            if (isPhoneTypeGsm()) {
                                newUnknownConnectionsGsm.add(mConnections[i]);
                            } else {
                                newUnknownConnectionCdma = mConnections[i];
                            }
                        }
                    }
                }
                hasNonHangupStateChanged = true;
            } else if (conn != null && dc == null) {

                /// M: CC
                if (DBG_POLL) log("case 2 : old Call disappear");

                if (isPhoneTypeGsm()) {

                    /// M: CC: Convert state from WAITING to INCOMING @{
                    //[ALPS00401290]
                    if (((conn.getCall() == mForegroundCall &&
                            mForegroundCall.mConnections.size() == 1 &&
                            mBackgroundCall.isIdle()) ||
                            (conn.getCall() == mBackgroundCall &&
                            mBackgroundCall.mConnections.size() == 1 &&
                            mForegroundCall.isIdle())) &&
                            mRingingCall.getState() == GsmCdmaCall.State.WAITING) {
                        mRingingCall.mState = GsmCdmaCall.State.INCOMING;
                    }
                    /// @}

                    // Connection missing in CLCC response that we were
                    // tracking.
                    mDroppedDuringPoll.add(conn);
                    // Dropped connections are removed from the CallTracker
                    // list but kept in the GsmCdmaCall list
                    mConnections[i] = null;

                    /// M: CC: Proprietary incoming call handling @{
                    mHelper.CallIndicationEnd();
                    /// @}

                    /// M: CC: Forwarding number via EAIC @{
                    //To clear forwarding address if needed
                    mHelper.clearForwardingAddressVariables(i);
                    /// @}
                } else {
                    // This case means the RIL has no more active call anymore and
                    // we need to clean up the foregroundCall and ringingCall.
                    // Loop through foreground call connections as
                    // it contains the known logical connections.
                    int count = mForegroundCall.mConnections.size();
                    for (int n = 0; n < count; n++) {
                        if (Phone.DEBUG_PHONE) log("adding fgCall cn " + n + " to droppedDuringPoll");
                        GsmCdmaConnection cn = (GsmCdmaConnection)mForegroundCall.mConnections.get(n);
                        mDroppedDuringPoll.add(cn);
                    }
                    count = mRingingCall.mConnections.size();
                    // Loop through ringing call connections as
                    // it may contain the known logical connections.
                    for (int n = 0; n < count; n++) {
                        if (Phone.DEBUG_PHONE) log("adding rgCall cn " + n + " to droppedDuringPoll");
                        GsmCdmaConnection cn = (GsmCdmaConnection)mRingingCall.mConnections.get(n);
                        mDroppedDuringPoll.add(cn);
                    }

                    // Re-start Ecm timer when the connected emergency call ends
                    if (mIsEcmTimerCanceled) {
                        handleEcmTimer(GsmCdmaPhone.RESTART_ECM_TIMER);
                    }
                    // If emergency call is not going through while dialing
                    checkAndEnableDataCallAfterEmergencyCallDropped();

                    // Dropped connections are removed from the CallTracker
                    // list but kept in the Call list
                    mConnections[i] = null;

                }
            } else if (conn != null && dc != null && !conn.compareTo(dc) && isPhoneTypeGsm()) {

                /// M: CC
                if (DBG_POLL) log("case 3 : old Call replaced");

                // Connection in CLCC response does not match what
                // we were tracking. Assume dropped call and new call

                mDroppedDuringPoll.add(conn);

                /// M: CC: Fix AOSP bug - to clear mPendingMO @{
                if (mPendingMO != null && mPendingMO.compareTo(dc)) {
                    log("ringing disc not updated yet & replaced by pendingMo");

                    /// M: CC: For 3G VT only @{
                    // MO:set id to VT service
                    if (SystemProperties.get("ro.mtk_vt3g324m_support").equals("1")) {
                        if ((mForegroundCall.mVTProvider != null) && dc.isVideo) {
                            mForegroundCall.mVTProvider.setId(i + 1);
                        }
                    }
                    /// @}
                    mConnections[i] = mPendingMO;
                    mPendingMO.mIndex = i;
                    mPendingMO.update(dc);
                    mPendingMO = null;
                } else {
                    mConnections[i] = new GsmCdmaConnection (mPhone, dc, this, i);
                }
                //mConnections[i] = new GsmCdmaConnection (mPhone, dc, this, i);
                /// @}

                if (mConnections[i].getCall() == mRingingCall) {
                    newRinging = mConnections[i];
                } // else something strange happened
                hasNonHangupStateChanged = true;
            } else if (conn != null && dc != null) { /* implicit conn.compareTo(dc) */
                // Call collision case
                if (!isPhoneTypeGsm() && conn.isIncoming() != dc.isMT) {
                    if (dc.isMT == true) {
                        // Mt call takes precedence than Mo,drops Mo
                        mDroppedDuringPoll.add(conn);
                        // find if the MT call is a new ring or unknown connection
                        newRinging = checkMtFindNewRinging(dc,i);
                        if (newRinging == null) {
                            unknownConnectionAppeared = true;
                            newUnknownConnectionCdma = conn;
                        }
                        checkAndEnableDataCallAfterEmergencyCallDropped();
                    } else {
                        // Call info stored in conn is not consistent with the call info from dc.
                        // We should follow the rule of MT calls taking precedence over MO calls
                        // when there is conflict, so here we drop the call info from dc and
                        // continue to use the call info from conn, and only take a log.
                        Rlog.e(LOG_TAG,"Error in RIL, Phantom call appeared " + dc);
                    }
                } else {

                    /// M: CC
                    if (DBG_POLL) log("case 4 : old Call update");

                    boolean changed;
                    changed = conn.update(dc);
                    hasNonHangupStateChanged = hasNonHangupStateChanged || changed;
                }
            }

            if (REPEAT_POLLING) {
                if (dc != null) {
                    // FIXME with RIL, we should not need this anymore
                    if ((dc.state == DriverCall.State.DIALING
                            /*&& cm.getOption(cm.OPTION_POLL_DIALING)*/)
                        || (dc.state == DriverCall.State.ALERTING
                            /*&& cm.getOption(cm.OPTION_POLL_ALERTING)*/)
                        || (dc.state == DriverCall.State.INCOMING
                            /*&& cm.getOption(cm.OPTION_POLL_INCOMING)*/)
                        || (dc.state == DriverCall.State.WAITING
                            /*&& cm.getOption(cm.OPTION_POLL_WAITING)*/)) {
                        // Sometimes there's no unsolicited notification
                        // for state transitions
                        needsPollDelay = true;
                    }
                }
            }
        }

        // Safety check so that obj is not stuck with mIsInEmergencyCall set to true (and data
        // disabled). This should never happen though.
        if (!isPhoneTypeGsm() && noConnectionExists) {
            checkAndEnableDataCallAfterEmergencyCallDropped();
        }

        // This is the first poll after an ATD.
        // We expect the pending call to appear in the list
        // If it does not, we land here
        if (mPendingMO != null) {
            Rlog.d(LOG_TAG, "Pending MO dropped before poll fg state:"
                    + mForegroundCall.getState());

            mDroppedDuringPoll.add(mPendingMO);
            mPendingMO = null;
            mHangupPendingMO = false;

            if (!isPhoneTypeGsm()) {
                if( mPendingCallInEcm) {
                    mPendingCallInEcm = false;
                }
                checkAndEnableDataCallAfterEmergencyCallDropped();
            }
        }

        if (newRinging != null) {
            mPhone.notifyNewRingingConnection(newRinging);

            /// M: CC: For 3G VT only @{
            //MT:new VT service
            if (SystemProperties.get("ro.mtk_vt3g324m_support").equals("1")) {
                if (newRinging.isVideo()) {
                    newRinging.setVideoState(VideoProfile.STATE_BIDIRECTIONAL);
                    try {
                        mRingingCall.mVTProvider = new GsmVTProvider(
                                ((GsmCdmaConnection) newRinging).getGsmCdmaIndex());
                        log("handlePollCalls new GsmVTProvider");
                        IGsmVideoCallProvider gsmVideoCallProvider =
                                mRingingCall.mVTProvider.getInterface();
                        if (gsmVideoCallProvider != null) {
                            GsmVideoCallProviderWrapper gsmVideoCallProviderWrapper =
                                    new GsmVideoCallProviderWrapper(gsmVideoCallProvider);
                            log("handlePollCalls new GsmVideoCallProviderWrapper");
                            newRinging.setVideoProvider(gsmVideoCallProviderWrapper);
                        }
                    } catch (CallStateException ex) {
                    } catch (ClassCastException e) {
                        Rlog.e(LOG_TAG, "cast to GsmCdmaConnection fail for newRinging " + e);
                    } catch (RemoteException e) {
                        Rlog.e(LOG_TAG, "handlePollCalls new GsmVideoCallProviderWrapper failed");
                    }
                }
            }
            /// @}
        }


        // clear the "local hangup" and "missed/rejected call"
        // cases from the "dropped during poll" list
        // These cases need no "last call fail" reason
        for (int i = mDroppedDuringPoll.size() - 1; i >= 0 ; i--) {
            GsmCdmaConnection conn = mDroppedDuringPoll.get(i);
            //CDMA
            boolean wasDisconnected = false;

            /// M: CC: Modem reset related handling @{
            if (isCommandExceptionRadioNotAvailable(ar.exception)) {
                mDroppedDuringPoll.remove(i);
                hasAnyCallDisconnected |= conn.onDisconnect(DisconnectCause.LOST_SIGNAL);
                wasDisconnected = true;
            /// @}
            } else if (conn.isIncoming() && conn.getConnectTime() == 0) {
                // Missed or rejected call
                int cause;
                if (conn.mCause == DisconnectCause.LOCAL) {
                    cause = DisconnectCause.INCOMING_REJECTED;
                } else {
                    cause = DisconnectCause.INCOMING_MISSED;
                }

                if (Phone.DEBUG_PHONE) {
                    log("missed/rejected call, conn.cause=" + conn.mCause);
                    log("setting cause to " + cause);
                }
                mDroppedDuringPoll.remove(i);
                hasAnyCallDisconnected |= conn.onDisconnect(cause);
                wasDisconnected = true;
            } else if (conn.mCause == DisconnectCause.LOCAL
                    || conn.mCause == DisconnectCause.INVALID_NUMBER) {
                mDroppedDuringPoll.remove(i);
                hasAnyCallDisconnected |= conn.onDisconnect(conn.mCause);
                wasDisconnected = true;
            }

            if (!isPhoneTypeGsm() && wasDisconnected && unknownConnectionAppeared
                    && conn == newUnknownConnectionCdma) {
                unknownConnectionAppeared = false;
                newUnknownConnectionCdma = null;
            }
        }

        /// M: Added method to fulfill conference SRVCC for the host side. @{
        // The conference participant side is handled as normal call SRVCC.
        if (mImsConfHostConnection != null) {
            ImsPhoneConnection hostConn = (ImsPhoneConnection) mImsConfHostConnection;
            if (mImsConfParticipants.size() >= 2) {
                // Participants >= 2, apply MTK SRVCC solution.

                // Try to restore participants' address, we don't sure if +ECONFSRVCC is arrival.
                restoreConferenceParticipantAddress();

                log("SRVCC: notify new participant connections");
                hostConn.notifyConferenceConnectionsConfigured(mImsConfParticipants);
            } else if (mImsConfParticipants.size() == 1) {
                // Participants = 1, can't be a conference, so apply Google SRVCC solution.
                GsmCdmaConnection participant = (GsmCdmaConnection) mImsConfParticipants.get(0);

                // Conference host side with only one participant case.
                // Due to modem's limitation, we still need to restore the address since modem
                // doesn't notify address information.
                String address = hostConn.getConferenceParticipantAddress(0);
                log("SRVCC: restore participant connection with address: " + address);
                participant.updateConferenceParticipantAddress(address);

                log("SRVCC: only one connection, consider it as a normal call SRVCC");
                mPhone.notifyHandoverStateChanged(participant);
            } else {
                Rlog.e(LOG_TAG, "SRVCC: abnormal case, no participant connections.");
            }
            mImsConfParticipants.clear();
            mImsConfHostConnection = null;
            mEconfSrvccConnectionIds = null;
        }
        /// @}

        /* Disconnect any pending Handover connections */
        for (Iterator<Connection> it = mHandoverConnections.iterator();
                it.hasNext();) {
            Connection hoConnection = it.next();
            log("handlePollCalls - disconnect hoConn= " + hoConnection +
                    " hoConn.State= " + hoConnection.getState());
            if (hoConnection.getState().isRinging()) {
                hoConnection.onDisconnect(DisconnectCause.INCOMING_MISSED);
            } else {
                hoConnection.onDisconnect(DisconnectCause.NOT_VALID);
            }
            it.remove();
        }

        // Any non-local disconnects: determine cause
        /// M: CC: For 3G VT only @{
        //if (mDroppedDuringPoll.size() > 0) {
        if (mDroppedDuringPoll.size() > 0 &&
                !hasPendingReplaceRequest) {
        /// @}
            mCi.getLastCallFailCause(
                obtainNoPollCompleteMessage(EVENT_GET_LAST_CALL_FAIL_CAUSE));
        }

        if (needsPollDelay) {
            pollCallsAfterDelay();
        }

        // Cases when we can no longer keep disconnected Connection's
        // with their previous calls
        // 1) the phone has started to ring
        // 2) A Call/Connection object has changed state...
        //    we may have switched or held or answered (but not hung up)
        if ((newRinging != null || hasNonHangupStateChanged || hasAnyCallDisconnected)
            /// M: CC: Proprietary CRSS handling @{
            && !mHasPendingSwapRequest) {
            /// @}
            internalClearDisconnected();
        }

        if (VDBG) log("handlePollCalls calling updatePhoneState()");
        updatePhoneState();

        if (unknownConnectionAppeared) {
            if (isPhoneTypeGsm()) {
                for (Connection c : newUnknownConnectionsGsm) {
                    log("Notify unknown for " + c);
                    mPhone.notifyUnknownConnection(c);
                }
            } else {
                mPhone.notifyUnknownConnection(newUnknownConnectionCdma);
            }
        }

        if (hasNonHangupStateChanged || newRinging != null || hasAnyCallDisconnected) {
            mPhone.notifyPreciseCallStateChanged();
        }

        /// M: CC: Convert state from WAITING to INCOMING @{
        //[ALPS00401290]
        if (isPhoneTypeGsm() && mConnections != null
                && mConnections.length == MAX_CONNECTIONS_GSM) {
            if ((mHelper.getCurrentTotalConnections() == 1) &&
                    (mRingingCall.getState() == GsmCdmaCall.State.WAITING)) {
                mRingingCall.mState = GsmCdmaCall.State.INCOMING;
            }
        }
        /// @}

        //dumpState();
    }

    private void handleRadioNotAvailable() {
        // handlePollCalls will clear out its
        // call list when it gets the CommandException
        // error result from this
        pollCallsWhenSafe();
    }

    private void dumpState() {
        List l;

        Rlog.i(LOG_TAG,"Phone State:" + mState);

        Rlog.i(LOG_TAG,"Ringing call: " + mRingingCall.toString());

        l = mRingingCall.getConnections();
        for (int i = 0, s = l.size(); i < s; i++) {
            Rlog.i(LOG_TAG,l.get(i).toString());
        }

        Rlog.i(LOG_TAG,"Foreground call: " + mForegroundCall.toString());

        l = mForegroundCall.getConnections();
        for (int i = 0, s = l.size(); i < s; i++) {
            Rlog.i(LOG_TAG,l.get(i).toString());
        }

        Rlog.i(LOG_TAG,"Background call: " + mBackgroundCall.toString());

        l = mBackgroundCall.getConnections();
        for (int i = 0, s = l.size(); i < s; i++) {
            Rlog.i(LOG_TAG,l.get(i).toString());
        }

        /// M: CC: Use GsmCallTrackerHelper @{
        if (isPhoneTypeGsm()) {
            mHelper.LogState();
        }
        // @}
    }

    //***** Called from GsmCdmaConnection

    public void hangup(GsmCdmaConnection conn) throws CallStateException {
        if (conn.mOwner != this) {
            throw new CallStateException ("GsmCdmaConnection " + conn
                                    + "does not belong to GsmCdmaCallTracker " + this);
        }

        if (conn == mPendingMO) {
            // We're hanging up an outgoing call that doesn't have it's
            // GsmCdma index assigned yet

            if (Phone.DEBUG_PHONE) log("hangup: set hangupPendingMO to true");
            mHangupPendingMO = true;

            /// M: CC: Hangup special handling @{
            mHelper.PendingHangupRequestReset();
            /// @}
        } else if (!isPhoneTypeGsm()
                && conn.getCall() == mRingingCall
                && mRingingCall.getState() == GsmCdmaCall.State.WAITING) {
            /// M: CDMA log @{
            log("hangup waiting call");
            /// @}

            // Handle call waiting hang up case.
            //
            // The ringingCall state will change to IDLE in GsmCdmaCall.detach
            // if the ringing call connection size is 0. We don't specifically
            // set the ringing call state to IDLE here to avoid a race condition
            // where a new call waiting could get a hang up from an old call
            // waiting ringingCall.
            //
            // PhoneApp does the call log itself since only PhoneApp knows
            // the hangup reason is user ignoring or timing out. So conn.onDisconnect()
            // is not called here. Instead, conn.onLocalDisconnect() is called.
            conn.onLocalDisconnect();

            updatePhoneState();
            mPhone.notifyPreciseCallStateChanged();
            return;
        } else {
            try {
                /// M: CC: Hangup special handling @{
                //mCi.hangupConnection (conn.getGsmCdmaIndex(), obtainCompleteMessage());
                mCi.hangupConnection(conn.getGsmCdmaIndex(),
                        obtainCompleteMessage(EVENT_HANG_UP_RESULT));
                /// @}
            } catch (CallStateException ex) {
                // Ignore "connection not found"
                // Call may have hung up already
                /// M: CC: Hangup special handling @{
                mHelper.PendingHangupRequestReset();
                /// @}
                Rlog.w(LOG_TAG,"GsmCdmaCallTracker WARN: hangup() on absent connection "
                                + conn);
            }
        }

        conn.onHangupLocal();
    }

    public void separate(GsmCdmaConnection conn) throws CallStateException {
        if (conn.mOwner != this) {
            throw new CallStateException ("GsmCdmaConnection " + conn
                                    + "does not belong to GsmCdmaCallTracker " + this);
        }
        try {
            mCi.separateConnection (conn.getGsmCdmaIndex(),
                obtainCompleteMessage(EVENT_SEPARATE_RESULT));
        } catch (CallStateException ex) {
            // Ignore "connection not found"
            // Call may have hung up already
            Rlog.w(LOG_TAG,"GsmCdmaCallTracker WARN: separate() on absent connection " + conn);
        }
    }

    //***** Called from GsmCdmaPhone

    public void setMute(boolean mute) {
        mDesiredMute = mute;
        mCi.setMute(mDesiredMute, null);
    }

    public boolean getMute() {
        return mDesiredMute;
    }


    //***** Called from GsmCdmaCall

    public void hangup(GsmCdmaCall call) throws CallStateException {
        if (call.getConnections().size() == 0) {
            throw new CallStateException("no connections in call");
        }

        /// M: CC: Hangup special handling @{
        if (mHelper.hasPendingHangupRequest()) {
            log("hangup(GsmCall) hasPendingHangupRequest = true");
            if (mHelper.ForceReleaseAllConnection(call)) {
                return;
            }
        }
        /// @}

        if (call == mRingingCall) {
            if (Phone.DEBUG_PHONE) log("(ringing) hangup waiting or background");
            /// M: CC: Hangup special handling @{
            //mCi.hangupWaitingOrBackground(obtainCompleteMessage());
            mHelper.PendingHangupRequestInc();
            hangup((GsmCdmaConnection) (call.getConnections().get(0)));
            /// @}
        } else if (call == mForegroundCall) {
            /// M: CC: Hangup special handling @{
            mHelper.PendingHangupRequestInc();
            /// @}
            if (call.isDialingOrAlerting()) {
                if (Phone.DEBUG_PHONE) {
                    log("(foregnd) hangup dialing or alerting...");
                }
                hangup((GsmCdmaConnection)(call.getConnections().get(0)));
            /*
            /// M: CC: Use 1+SEND MMI to release active calls & accept held or waiting call @{
            // [ALPS02087255] [Call]MMI 1 cannot accept the waiting call.
            // 3GPP 22.030 6.5.5
            // "Releases all active calls (if any exist) and accepts
            //  the other (held or waiting) call."
            } else if (isPhoneTypeGsm()
                    && mRingingCall.isRinging()) {
                // Do not auto-answer ringing on CHUP, instead just end active calls
                log("hangup all conns in active/background call, without affecting ringing call");
                hangupAllConnections(call);
            */
            } else {
                /// M: CC: Can not end the ECC call when enable SIM PIN lock
                //[ALPS01431282][ALPS.KK1.MP2.V2.4 Regression Test]
                //hangupForegroundResumeBackground();
                if (Phone.DEBUG_PHONE) log("(foregnd) hangup active");
                if (isPhoneTypeGsm()) {
                    GsmCdmaConnection cn = (GsmCdmaConnection) call.getConnections().get(0);
                    String address = cn.getAddress();
                    /// M: CC: ECC Retry @{
                    if (PhoneNumberUtils.isEmergencyNumber(mPhone.getSubId(), address) &&
                    /// @}
                            !PhoneNumberUtils.isSpecialEmergencyNumber(address)) {
                        log("(foregnd) hangup active Emergency call by connection index");
                        hangup((GsmCdmaConnection) (call.getConnections().get(0)));
                    } else {
                        hangupForegroundResumeBackground();
                    }
                } else {
                    hangupForegroundResumeBackground();
                }
            }
        } else if (call == mBackgroundCall) {
            if (mRingingCall.isRinging()) {
                if (Phone.DEBUG_PHONE) {
                    log("hangup all conns in background call");
                }
                hangupAllConnections(call);
            } else {
                /// M: CC: Hangup special handling @{
                mHelper.PendingHangupRequestInc();
                if (Phone.DEBUG_PHONE) log("(backgnd) hangup waiting/background");
                /// @}
                hangupWaitingOrBackground();
            }
        } else {
            throw new RuntimeException ("GsmCdmaCall " + call +
                    "does not belong to GsmCdmaCallTracker " + this);
        }

        call.onHangupLocal();
        mPhone.notifyPreciseCallStateChanged();
    }

    public void hangupWaitingOrBackground() {
        if (Phone.DEBUG_PHONE) log("hangupWaitingOrBackground");
        /// M: CC: Hangup special handling @{
        //mCi.hangupWaitingOrBackground(obtainCompleteMessage());
        mCi.hangupWaitingOrBackground(obtainCompleteMessage(EVENT_HANG_UP_RESULT));
        /// @}
    }

    public void hangupForegroundResumeBackground() {
        if (Phone.DEBUG_PHONE) log("hangupForegroundResumeBackground");
        /// M: CC: Hangup special handling @{
        //mCi.hangupForegroundResumeBackground(obtainCompleteMessage());
        mCi.hangupForegroundResumeBackground(obtainCompleteMessage(EVENT_HANG_UP_RESULT));
        /// @}
    }

    public void hangupConnectionByIndex(GsmCdmaCall call, int index)
            throws CallStateException {
        int count = call.mConnections.size();
        for (int i = 0; i < count; i++) {
            GsmCdmaConnection cn = (GsmCdmaConnection)call.mConnections.get(i);

            /// M: CC: Hangup Conference special handling @{
            // Not to hangup a DISCONNECTED connection in a conference call
            // [ALPS02347071][ALPS02092424] GsmCall.mConnections NOT removed in handlePollCalls().
            if (cn.getState() == GsmCdmaCall.State.DISCONNECTED) {
                if (Phone.DEBUG_PHONE) log("hangupConnectionByIndex: hangup a DISCONNECTED conn");
                continue;
            }
            /// @}

            if (cn.getGsmCdmaIndex() == index) {
                mCi.hangupConnection(index, obtainCompleteMessage());
                return;
            }
        }

        /// M: CC: Hangup special handling @{
        if (mHelper.hangupBgConnectionByIndex(index))
            return;
        if (mHelper.hangupRingingConnectionByIndex(index))
            return;
        /// @}

        throw new CallStateException("no GsmCdma index found");
    }

    public void hangupAllConnections(GsmCdmaCall call) {
        try {
            int count = call.mConnections.size();
            for (int i = 0; i < count; i++) {
                GsmCdmaConnection cn = (GsmCdmaConnection)call.mConnections.get(i);
                mCi.hangupConnection(cn.getGsmCdmaIndex(), obtainCompleteMessage());
            }
        } catch (CallStateException ex) {
            Rlog.e(LOG_TAG, "hangupConnectionByIndex caught " + ex);
        }
    }

    public GsmCdmaConnection getConnectionByIndex(GsmCdmaCall call, int index)
            throws CallStateException {
        int count = call.mConnections.size();
        for (int i = 0; i < count; i++) {
            GsmCdmaConnection cn = (GsmCdmaConnection)call.mConnections.get(i);
            if (cn.getGsmCdmaIndex() == index) {
                return cn;
            }
        }

        return null;
    }

    //CDMA
    private void notifyCallWaitingInfo(CdmaCallWaitingNotification obj) {
        if (mCallWaitingRegistrants != null) {
            mCallWaitingRegistrants.notifyRegistrants(new AsyncResult(null, obj, null));
        }
    }

    //CDMA
    private void handleCallWaitingInfo(CdmaCallWaitingNotification cw) {
        /// M: CDMA process plus code @{
        processPlusCodeForWaitingCall(cw);
        /// @}
        /// M: CDMA process waiting call @{
        if (!shouldNotifyWaitingCall(cw)) {
            return;
        }
        /// @}

        // Create a new GsmCdmaConnection which attaches itself to ringingCall.
        new GsmCdmaConnection(mPhone.getContext(), cw, this, mRingingCall);
        updatePhoneState();

        // Finally notify application
        notifyCallWaitingInfo(cw);
    }

    private Phone.SuppService getFailedService(int what) {
        switch (what) {
            case EVENT_SWITCH_RESULT:
                return Phone.SuppService.SWITCH;
            case EVENT_CONFERENCE_RESULT:
                return Phone.SuppService.CONFERENCE;
            case EVENT_SEPARATE_RESULT:
                return Phone.SuppService.SEPARATE;
            case EVENT_ECT_RESULT:
                return Phone.SuppService.TRANSFER;
        }
        return Phone.SuppService.UNKNOWN;
    }

    //****** Overridden from Handler

    @Override
    public void handleMessage(Message msg) {
        AsyncResult ar;

        /// M: CC: Use GsmCallTrackerHelper @{
        mHelper.LogerMessage(msg.what);
        /// @}

        switch (msg.what) {
            case EVENT_POLL_CALLS_RESULT:
                Rlog.d(LOG_TAG, "Event EVENT_POLL_CALLS_RESULT Received");

                if (msg == mLastRelevantPoll) {
                    if (DBG_POLL) log(
                            "handle EVENT_POLL_CALL_RESULT: set needsPoll=F");
                    mNeedsPoll = false;
                    mLastRelevantPoll = null;
                    handlePollCalls((AsyncResult)msg.obj);
                }
            break;

            case EVENT_OPERATION_COMPLETE:
                operationComplete();
                /// M: CC: For 3G VT only @{
                if (hasPendingReplaceRequest) {
                    hasPendingReplaceRequest = false;
                }
                /// @}
            break;

            case EVENT_CONFERENCE_RESULT:
                if (isPhoneTypeGsm()) {
                    // The conference merge failed, so notify listeners.  Ultimately this bubbles up
                    // to Telecom, which will inform the InCall UI of the failure.
                    Connection connection = mForegroundCall.getLatestConnection();
                    if (connection != null) {
                        connection.onConferenceMergeFailed();
                    }
                }
                // fall through
            case EVENT_SEPARATE_RESULT:
            case EVENT_ECT_RESULT:
            /// M: CC: Hangup special handling @{
                if (isPhoneTypeGsm()) {
                    ar = (AsyncResult) msg.obj;
                    if (ar.exception != null) {
                        mHelper.PendingHangupRequestUpdate();
                        mPhone.notifySuppServiceFailed(getFailedService(msg.what));
                    }
                    operationComplete();
                } else {
                    throw new RuntimeException("unexpected event " + msg.what + " not handled by " +
                           "phone type " + mPhone.getPhoneType());
                }
            break;
            /// @}
            case EVENT_SWITCH_RESULT:
                if (isPhoneTypeGsm()) {
                    ar = (AsyncResult) msg.obj;
                    if (ar.exception != null) {
                        /// M: CC: Proprietary CRSS handling @{
                        if (mWaitingForHoldRequest.isWaiting()) {
                            mPendingMO.mCause = DisconnectCause.LOCAL;
                            mPendingMO.onDisconnect(DisconnectCause.LOCAL);
                            mPendingMO = null;
                            mHangupPendingMO = false;
                            updatePhoneState();

                            resumeBackgroundAfterDialFailed();
                            mWaitingForHoldRequest.reset();
                        }
                        /// @}
                        mPhone.notifySuppServiceFailed(getFailedService(msg.what));
                    /// M: CC: Proprietary CRSS handling @{
                    } else {
                        if (mWaitingForHoldRequest.isWaiting()) {
                            Rlog.i(LOG_TAG, "Switch success, and then dial");
                            mWaitingForHoldRequest.handleOperation();
                        }
                    }
                    mHasPendingSwapRequest = false;
                    /// @}
                    operationComplete();
                /// M: CC: Proprietary CRSS handling @{
                /*
                } else {
                    if (msg.what != EVENT_SWITCH_RESULT) {
                        // EVENT_SWITCH_RESULT in GSM call triggers operationComplete() which gets
                        // the current call list. But in CDMA there is no list so there is nothing
                        // to do. Other messages however are not expected in CDMA.
                        throw new RuntimeException("unexpected event " + msg.what + " not handled by " +
                                "phone type " + mPhone.getPhoneType());
                    }
                */
                }
            break;

            case EVENT_GET_LAST_CALL_FAIL_CAUSE:
                int causeCode;
                String vendorCause = null;
                ar = (AsyncResult)msg.obj;

                operationComplete();

                if (ar.exception != null) {
                    // An exception occurred...just treat the disconnect
                    // cause as "normal"
                    causeCode = CallFailCause.NORMAL_CLEARING;
                    Rlog.i(LOG_TAG,
                            "Exception during getLastCallFailCause, assuming normal disconnect");
                } else {
                    LastCallFailCause failCause = (LastCallFailCause)ar.result;
                    causeCode = failCause.causeCode;
                    vendorCause = failCause.vendorCause;
                }
                // Log the causeCode if its not normal
                if (causeCode == CallFailCause.NO_CIRCUIT_AVAIL ||
                    causeCode == CallFailCause.TEMPORARY_FAILURE ||
                    causeCode == CallFailCause.SWITCHING_CONGESTION ||
                    causeCode == CallFailCause.CHANNEL_NOT_AVAIL ||
                    causeCode == CallFailCause.QOS_NOT_AVAIL ||
                    causeCode == CallFailCause.BEARER_NOT_AVAIL ||
                    causeCode == CallFailCause.ERROR_UNSPECIFIED) {

                    CellLocation loc = mPhone.getCellLocation();
                    int cid = -1;
                    if (loc != null) {
                        if (isPhoneTypeGsm()) {
                            cid = ((GsmCellLocation)loc).getCid();
                        } else {
                            cid = ((CdmaCellLocation)loc).getBaseStationId();
                        }
                    }
                    EventLog.writeEvent(EventLogTags.CALL_DROP, causeCode, cid,
                            /// M: CC: DSDS bug fix @{
                            // getNetworkType with subId for GsmCellLocation
                            //TelephonyManager.getDefault().getNetworkType());
                            TelephonyManager.getDefault().getNetworkType(mPhone.getSubId()));
                            /// @}
                }

                for (int i = 0, s = mDroppedDuringPoll.size(); i < s ; i++) {
                    GsmCdmaConnection conn = mDroppedDuringPoll.get(i);

                    conn.onRemoteDisconnect(causeCode, vendorCause);
                }

                updatePhoneState();

                mPhone.notifyPreciseCallStateChanged();
                mDroppedDuringPoll.clear();
            break;

            case EVENT_REPOLL_AFTER_DELAY:
            case EVENT_CALL_STATE_CHANGE:
                pollCallsWhenSafe();
            break;

            case EVENT_RADIO_AVAILABLE:
                handleRadioAvailable();
            break;

            case EVENT_RADIO_NOT_AVAILABLE:
                handleRadioNotAvailable();
            break;

            case EVENT_EXIT_ECM_RESPONSE_CDMA:
                if (!isPhoneTypeGsm()) {
                    // no matter the result, we still do the same here
                    if (mPendingCallInEcm) {
                        /// M: CDMA process plus code @{
                        final String dialString = (String) ((AsyncResult) msg.obj).userObj;
                        /// @}
                        /// M: CDMA FDN @{
                        String tmpStr = mPendingMO.getAddress() + ","
                                + PhoneNumberUtils.extractNetworkPortionAlt(dialString);
                        /// @}
                        mCi.dial(tmpStr, mPendingCallClirMode, obtainCompleteMessage());
                        mPendingCallInEcm = false;
                        /// M: CDMA process plus code @{
                        if (mCdmaCallTrackerExt != null && mCdmaCallTrackerExt.needToConvert(
                                dialString, GsmCdmaConnection.formatDialString(dialString))) {
                            mPendingMO.setConverted(
                                    PhoneNumberUtils.extractNetworkPortionAlt(dialString));
                        }
                        /// @}
                    }
                    mPhone.unsetOnEcbModeExitResponse(this);
                } else {
                    throw new RuntimeException("unexpected event " + msg.what + " not handled by " +
                            "phone type " + mPhone.getPhoneType());
                }
                break;

            case EVENT_CALL_WAITING_INFO_CDMA:
                if (!isPhoneTypeGsm()) {
                    ar = (AsyncResult)msg.obj;
                    if (ar.exception == null) {
                        handleCallWaitingInfo((CdmaCallWaitingNotification)ar.result);
                        Rlog.d(LOG_TAG, "Event EVENT_CALL_WAITING_INFO_CDMA Received");
                    }
                } else {
                    throw new RuntimeException("unexpected event " + msg.what + " not handled by " +
                            "phone type " + mPhone.getPhoneType());
                }
                break;

            case EVENT_THREE_WAY_DIAL_L2_RESULT_CDMA:
                if (!isPhoneTypeGsm()) {
                    ar = (AsyncResult)msg.obj;
                    if (ar.exception == null) {
                        // Assume 3 way call is connected
                        /// M: CDMA process 3 way dialing @{
                        if (mPendingMO != null) {
                            mPendingMO.onConnectedInOrOut();
                        }
                        /// @}
                        mPendingMO = null;
                    }
                } else {
                    throw new RuntimeException("unexpected event " + msg.what + " not handled by " +
                            "phone type " + mPhone.getPhoneType());
                }
                break;

            case EVENT_THREE_WAY_DIAL_BLANK_FLASH:
                if (!isPhoneTypeGsm()) {
                    ar = (AsyncResult) msg.obj;
                    if (ar.exception == null) {
                        /// M: CDMA process plus code @{
                        final String dialString = (String) ((AsyncResult) msg.obj).userObj;
                        /// @}
                        postDelayed(
                                new Runnable() {
                                    public void run() {
                                        if (mPendingMO != null) {
                                            /// M: CDMA FDN @{
                                            String tmpStr = mPendingMO.getAddress() + ","
                                                    + PhoneNumberUtils.extractNetworkPortionAlt(
                                                    dialString);
                                            /// @}
                                            mCi.sendCDMAFeatureCode(tmpStr,
                                                    obtainMessage(
                                                    EVENT_THREE_WAY_DIAL_L2_RESULT_CDMA));
                                            /// M: CDMA process plus code @{
                                            if (mCdmaCallTrackerExt != null &&
                                                    mCdmaCallTrackerExt.needToConvert(dialString,
                                                    GsmCdmaConnection.formatDialString(
                                                    dialString))) {
                                                mPendingMO.setConverted(
                                                        PhoneNumberUtils.extractNetworkPortionAlt(
                                                        dialString));
                                            }
                                            /// @}
                                        }
                                    }
                                }, m3WayCallFlashDelay);
                    } else {
                        mPendingMO = null;
                        Rlog.w(LOG_TAG, "exception happened on Blank Flash for 3-way call");
                    }
                } else {
                    throw new RuntimeException("unexpected event " + msg.what + " not handled by " +
                            "phone type " + mPhone.getPhoneType());
                }
                break;

            /// M: CC: Hangup special handling @{
            case EVENT_HANG_UP_RESULT:
                mHelper.PendingHangupRequestDec();
                operationComplete();
            break;

            case EVENT_DIAL_CALL_RESULT:
                ar = (AsyncResult) msg.obj;
                if (ar.exception != null) {
                    log("dial call failed!!");
                    mHelper.PendingHangupRequestUpdate();
                }
                operationComplete();
            break;
            /// @}

            /// M: CC: Proprietary incoming call handling @{
            case EVENT_INCOMING_CALL_INDICATION:
                mHelper.CallIndicationProcess((AsyncResult) msg.obj);
            break;
            /// @}

            /// M: CC: Modem reset related handling
            case EVENT_RADIO_OFF_OR_NOT_AVAILABLE:
                log("Receives EVENT_RADIO_OFF_OR_NOT_AVAILABLE");
                handlePollCalls(new AsyncResult(null, null,
                        new CommandException(CommandException.Error.RADIO_NOT_AVAILABLE)));
            break;
            /// @}

            ///M: For IMS conference SRVCC @{
            case EVENT_ECONF_SRVCC_INDICATION:
                log("Receives EVENT_ECONF_SRVCC_INDICATION");
                ar = (AsyncResult) msg.obj;
                mEconfSrvccConnectionIds = (int[]) ar.result;

                // Mark ECONFSRVCC is received, then do poll calls now.
                mNeedWaitImsEConfSrvcc = false;
                pollCallsWhenSafe();
            break;
            /// @}

            /// M: CDMA process call accepted @{
            case EVENT_CDMA_CALL_ACCEPTED:
                ar = (AsyncResult) msg.obj;
                if (ar.exception == null) {
                    handleCallAccepted();
                    log("EVENT_CDMA_CALL_ACCEPTED");
                }
               break;
            /// @}

            /// M: CC: Vzw ECC/hVoLTE redial @{
            case EVENT_CALL_REDIAL_STATE:
                log("Receives EVENT_CALL_REDIAL_STATE");
                ar = (AsyncResult) msg.obj;
                mRedialState = redialStateFromInt((int) ar.result);
            break;
            /// @}

            default:{
                throw new RuntimeException("unexpected event " + msg.what + " not handled by " +
                        "phone type " + mPhone.getPhoneType());
            }
        }
    }

    //CDMA
    /**
     * Check and enable data call after an emergency call is dropped if it's
     * not in ECM
     */
    private void checkAndEnableDataCallAfterEmergencyCallDropped() {
        if (mIsInEmergencyCall) {
            mIsInEmergencyCall = false;
            /// M: Get the property by phoneId @{
            //String inEcm=SystemProperties.get(TelephonyProperties.PROPERTY_INECM_MODE, "false");
            String inEcm = mPhone.getSystemProperty(
                    TelephonyProperties.PROPERTY_INECM_MODE, "false");
            /// @}
            if (Phone.DEBUG_PHONE) {
                log("checkAndEnableDataCallAfterEmergencyCallDropped,inEcm=" + inEcm);
            }
            if (inEcm.compareTo("false") == 0) {
                // Re-initiate data connection
                mPhone.mDcTracker.setInternalDataEnabled(true);
                mPhone.notifyEmergencyCallRegistrants(false);
            }
            mPhone.sendEmergencyCallStateChange(false);
        }
    }

    /**
     * Check the MT call to see if it's a new ring or
     * a unknown connection.
     */
    private Connection checkMtFindNewRinging(DriverCall dc, int i) {

        Connection newRinging = null;

        // it's a ringing call
        if (mConnections[i].getCall() == mRingingCall) {
            newRinging = mConnections[i];
            if (Phone.DEBUG_PHONE) log("Notify new ring " + dc);
        } else {
            // Something strange happened: a call which is neither
            // a ringing call nor the one we created. It could be the
            // call collision result from RIL
            Rlog.e(LOG_TAG,"Phantom call appeared " + dc);
            // If it's a connected call, set the connect time so that
            // it's non-zero.  It may not be accurate, but at least
            // it won't appear as a Missed Call.
            if (dc.state != DriverCall.State.ALERTING
                    && dc.state != DriverCall.State.DIALING) {
                mConnections[i].onConnectedInOrOut();
                if (dc.state == DriverCall.State.HOLDING) {
                    // We've transitioned into HOLDING
                    mConnections[i].onStartedHolding();
                }
            }
        }
        return newRinging;
    }

    //CDMA
    /**
     * Check if current call is in emergency call
     *
     * @return true if it is in emergency call
     *         false if it is not in emergency call
     */
    public boolean isInEmergencyCall() {
        return mIsInEmergencyCall;
    }

    private boolean isPhoneTypeGsm() {
        return mPhone.getPhoneType() == PhoneConstants.PHONE_TYPE_GSM;
    }

    public GsmCdmaPhone getPhone() {
        return mPhone;
    }

    @Override
    protected void log(String msg) {
        /// M: CC: Modify log
        //Rlog.d(LOG_TAG, "[GsmCdmaCallTracker] " + msg);
        Rlog.d(LOG_TAG, msg + " (slot " + mPhone.getPhoneId() + ")");
    }

    @Override
    public void dump(FileDescriptor fd, PrintWriter pw, String[] args) {
        pw.println("GsmCdmaCallTracker extends:");
        super.dump(fd, pw, args);
        pw.println("mConnections: length=" + mConnections.length);
        for(int i=0; i < mConnections.length; i++) {
            pw.printf("  mConnections[%d]=%s\n", i, mConnections[i]);
        }
        pw.println(" mVoiceCallEndedRegistrants=" + mVoiceCallEndedRegistrants);
        pw.println(" mVoiceCallStartedRegistrants=" + mVoiceCallStartedRegistrants);
        if (!isPhoneTypeGsm()) {
            pw.println(" mCallWaitingRegistrants=" + mCallWaitingRegistrants);
        }
        pw.println(" mDroppedDuringPoll: size=" + mDroppedDuringPoll.size());
        for(int i = 0; i < mDroppedDuringPoll.size(); i++) {
            pw.printf( "  mDroppedDuringPoll[%d]=%s\n", i, mDroppedDuringPoll.get(i));
        }
        pw.println(" mRingingCall=" + mRingingCall);
        pw.println(" mForegroundCall=" + mForegroundCall);
        pw.println(" mBackgroundCall=" + mBackgroundCall);
        pw.println(" mPendingMO=" + mPendingMO);
        pw.println(" mHangupPendingMO=" + mHangupPendingMO);
        pw.println(" mPhone=" + mPhone);
        pw.println(" mDesiredMute=" + mDesiredMute);
        pw.println(" mState=" + mState);
        if (!isPhoneTypeGsm()) {
            pw.println(" mPendingCallInEcm=" + mPendingCallInEcm);
            pw.println(" mIsInEmergencyCall=" + mIsInEmergencyCall);
            pw.println(" mPendingCallClirMode=" + mPendingCallClirMode);
            pw.println(" mIsEcmTimerCanceled=" + mIsEcmTimerCanceled);
        }

    }

    @Override
    public PhoneConstants.State getState() {
        return mState;
    }

    public int getMaxConnectionsPerCall() {
        return mPhone.isPhoneTypeGsm() ?
                MAX_CONNECTIONS_PER_CALL_GSM :
                MAX_CONNECTIONS_PER_CALL_CDMA;
    }

    /// M: CC: HangupAll for FTA 31.4.4.2 @{
    /**
     * used to release all connections in the MS,
     * release all connections with one reqeust together, not seperated.
     * @throws CallStateException if the callState is unexpected.
     */
    public void hangupAll() throws CallStateException {
        if (Phone.DEBUG_PHONE) log("hangupAll");
        mCi.hangupAll(obtainCompleteMessage());

        if (!mRingingCall.isIdle()) {
            mRingingCall.onHangupLocal();
        }
        if (!mForegroundCall.isIdle()) {
            mForegroundCall.onHangupLocal();
        }
        if (!mBackgroundCall.isIdle()) {
            mBackgroundCall.onHangupLocal();
        }
    }
    /// @}

    /// M: CC: Proprietary incoming call handling @{
    public void setIncomingCallIndicationResponse(boolean accept) {
        mHelper.CallIndicationResponse(accept);
    }

    public void registerForVoiceCallIncomingIndication(Handler h, int what, Object obj) {
        Registrant r = new Registrant(h, what, obj);
        mVoiceCallIncomingIndicationRegistrants.add(r);
    }

    public void unregisterForVoiceCallIncomingIndication(Handler h) {
        mVoiceCallIncomingIndicationRegistrants.remove(h);
    }
    /// @}

    /// M: CC: For 3G VT only @{
    private boolean canVtDial() {
        boolean ret;
        int networkType = mPhone.getServiceState().getVoiceNetworkType();
        log("networkType=" + TelephonyManager.getNetworkTypeName(networkType));

        ret = (networkType == TelephonyManager.NETWORK_TYPE_UMTS ||
                networkType == TelephonyManager.NETWORK_TYPE_HSDPA ||
                networkType == TelephonyManager.NETWORK_TYPE_HSUPA ||
                networkType == TelephonyManager.NETWORK_TYPE_HSPA ||
                networkType == TelephonyManager.NETWORK_TYPE_HSPAP ||
                networkType == TelephonyManager.NETWORK_TYPE_LTE);

        return ret;
    }

    /**
     * clirMode is one of the CLIR_ constants
     */
    public synchronized Connection vtDial(String dialString, int clirMode, UUSInfo uusInfo,
                                        Bundle intentExtras)
            throws CallStateException {
        // note that this triggers call state changed notif
        clearDisconnected();

        if (!canDial()) {
            throw new CallStateException("cannot dial in current state");
        }

        if (!canVtDial()) {
            throw new CallStateException("cannot vtDial under non 3/4G network");
        }

        String origNumber = dialString;
        dialString = convertNumberIfNecessary(mPhone, dialString);

        // Only one call can exist for 3G VT.
        // We assume CallsManager will block adding call if a 3G VT call has already existed.
        if (mForegroundCall.getState() != GsmCdmaCall.State.IDLE ||
            mRingingCall.getState().isRinging()) {
            //we should have failed in !canDial() above before we get here
            throw new CallStateException("cannot vtDial since non-IDLE call already exists");
        }

        mPendingMO = new GsmCdmaConnection(mPhone, checkForTestEmergencyNumber(dialString),
                this, mForegroundCall);
        mHangupPendingMO = false;
        mPendingMO.mIsVideo = true; //for 3G VT

        if ( mPendingMO.getAddress() == null || mPendingMO.getAddress().length() == 0
                || mPendingMO.getAddress().indexOf(PhoneNumberUtils.WILD) >= 0) {
            // Phone number is invalid
            mPendingMO.mCause = DisconnectCause.INVALID_NUMBER;

            // handlePollCalls() will notice this call not present
            // and will mark it as dropped.
            pollCallsWhenSafe();
        } else {
            // Always unmute when initiating a new call
            setMute(false);

            mCi.vtDial(mPendingMO.getAddress(), clirMode, uusInfo,
                    obtainCompleteMessage());

            mPendingMO.setVideoState(VideoProfile.STATE_BIDIRECTIONAL);

            //MO:new VT service
            mForegroundCall.mVTProvider = new GsmVTProvider();
            log("vtDial new GsmVTProvider");
            try {
                IGsmVideoCallProvider gsmVideoCallProvider =
                        mForegroundCall.mVTProvider.getInterface();
                if (gsmVideoCallProvider != null) {
                    GsmVideoCallProviderWrapper gsmVideoCallProviderWrapper =
                            new GsmVideoCallProviderWrapper(gsmVideoCallProvider);
                    log("vtDial new GsmVideoCallProviderWrapper");
                    mPendingMO.setVideoProvider(gsmVideoCallProviderWrapper);
                }
            } catch (RemoteException e) {
                Rlog.e(LOG_TAG, "vtDial new GsmVideoCallProviderWrapper failed");
            }


        }

        if (mNumberConverted) {
            mPendingMO.setConverted(origNumber);
            mNumberConverted = false;
        }

        updatePhoneState();
        mPhone.notifyPreciseCallStateChanged();

        return mPendingMO;
    }

    public Connection vtDial(String dialString, UUSInfo uusInfo, Bundle intentExtras)
            throws CallStateException {
        return vtDial(dialString, CommandsInterface.CLIR_DEFAULT, uusInfo, intentExtras);
    }

    public void acceptCall(int videoState) throws CallStateException {
        // FIXME if SWITCH fails, should retry with ANSWER
        // in case the active/holding call disappeared and this
        // is no longer call waiting

        if (mRingingCall.getState() == GsmCdmaCall.State.INCOMING) {
            Rlog.i("phone", "acceptCall: incoming...");
            // Always unmute when answering a new call
            setMute(false);
            /// M: For both VT and ViLTE @{
            GsmCdmaConnection cn = (GsmCdmaConnection) mRingingCall.mConnections.get(0);
            if (cn.isVideo()) {
                if (videoState == VideoProfile.STATE_AUDIO_ONLY) {
                    mCi.acceptVtCallWithVoiceOnly(cn.getGsmCdmaIndex(), obtainCompleteMessage());
                    cn.setVideoState(VideoProfile.STATE_AUDIO_ONLY);
                    return;
                }
            }
            /// @}
            mCi.acceptCall(obtainCompleteMessage());
        } else if (mRingingCall.getState() == GsmCdmaCall.State.WAITING) {
            if (isPhoneTypeGsm()) {
                setMute(false);
                /// M: For both VT and ViLTE @{
                GsmCdmaConnection cn = (GsmCdmaConnection) mRingingCall.mConnections.get(0);
                if (cn.isVideo()) {
                    GsmCdmaConnection fgCn =
                            (GsmCdmaConnection) mForegroundCall.mConnections.get(0);
                    if (fgCn != null && fgCn.isVideo()) {
                        hasPendingReplaceRequest = true;
                        mCi.replaceVtCall(fgCn.mIndex + 1, obtainCompleteMessage());
                        fgCn.onHangupLocal();
                        return;
                    }
                }
                /// @}
            } else {
                GsmCdmaConnection cwConn = (GsmCdmaConnection)(mRingingCall.getLatestConnection());

                // Since there is no network response for supplimentary
                // service for CDMA, we assume call waiting is answered.
                // ringing Call state change to idle is in GsmCdmaCall.detach
                // triggered by updateParent.
                cwConn.updateParent(mRingingCall, mForegroundCall);
                cwConn.onConnectedInOrOut();
                updatePhoneState();
            }
            switchWaitingOrHoldingAndActive();
        } else {
            throw new CallStateException("phone not ringing");
        }
    }
    /// @}

    /// M: CC: Proprietary CRSS handling @{
    private void resumeBackgroundAfterDialFailed() {
        // We need to make a copy here, since fakeHoldBeforeDial()
        // modifies the lists, and we don't want to reverse the order
        List<Connection> connCopy = (List<Connection>) mBackgroundCall.mConnections.clone();

        for (int i = 0, s = connCopy.size() ; i < s ; i++) {
            GsmCdmaConnection conn = (GsmCdmaConnection) connCopy.get(i);

            conn.resumeHoldAfterDialFailed();
        }
    }
    /// @}

    /// M: For IMS conference SRVCC. @{
    /**
     * For conference participants, the call number will be empty after SRVCC.
     * So at handlePollCalls(), it will get new connections without address. We use +ECONFSRVCC
     * and conference XML to restore all addresses.
     *
     * @return true if connections are restored.
     */
    private synchronized boolean restoreConferenceParticipantAddress() {
        if (mEconfSrvccConnectionIds == null) {
            log("SRVCC: restoreConferenceParticipantAddress():" +
                    "ignore because mEconfSrvccConnectionIds is empty");
            return false;
        }

        boolean finishRestore = false;

        // int[] mEconfSrvccConnectionIds = { size, call-ID-1, call-ID-2, call-ID-3, ...}
        int numOfParticipants = mEconfSrvccConnectionIds[0];
        for (int index = 1; index <= numOfParticipants; index++) {

            int participantCallId = mEconfSrvccConnectionIds[index];
            GsmCdmaConnection participantConnection = mConnections[participantCallId - 1];

            if (participantConnection != null) {
                log("SRVCC: found conference connections!");

                ImsPhoneConnection hostConnection = null;
                if (participantConnection.mOrigConnection instanceof ImsPhoneConnection) {
                    hostConnection = (ImsPhoneConnection) participantConnection.mOrigConnection;
                } else {
                    log("SRVCC: host is abnormal, ignore connection: " + participantConnection);
                    continue;
                }

                if (hostConnection == null) {
                    log("SRVCC: no host, ignore connection: " + participantConnection);
                    continue;
                }

                String address = hostConnection.getConferenceParticipantAddress(index - 1);
                participantConnection.updateConferenceParticipantAddress(address);
                finishRestore = true;

                log("SRVCC: restore Connection=" + participantConnection +
                        " with address:" + address);
            }
        }

        return finishRestore;
    }

    @Override
    protected Connection getHoConnection(DriverCall dc) {
        if (dc == null) {
            return null;
        }
        /*
         CallTracker.getHoConnection is used to find the original connection before SRVCC. It finds
         by call number and call state. But in conference SRVCC case, the call number is null and
         the call state might be different.
        */
        if (mEconfSrvccConnectionIds != null && dc != null) {
            int numOfParticipants = mEconfSrvccConnectionIds[0];
            for (int index = 1; index <= numOfParticipants; index++) {
                if (dc.index == mEconfSrvccConnectionIds[index]) {
                    log("SRVCC: getHoConnection for call-id:"
                            + dc.index + " in a conference is found!");
                    if (mImsConfHostConnection == null) {
                        log("SRVCC: but mImsConfHostConnection is null, try to find by callState");
                        break;
                    } else {
                        log("SRVCC: ret= " + mImsConfHostConnection);
                        return mImsConfHostConnection;
                    }
                }
            }
        }

        return super.getHoConnection(dc);
    }
    /// @}

    /// M: CDMA process plus code @{
    private void processPlusCodeForWaitingCall(CdmaCallWaitingNotification cw) {
        // Optimize callwaiting
        String address = cw.number;
        // Make sure there's a leading + on addresses with a TOA of 145
        log("processPlusCodeForWaitingCall before format number:" + cw.number);
        if (address != null && address.length() > 0 && mCdmaCallTrackerExt != null) {
            cw.number = mCdmaCallTrackerExt.processPlusCodeForWaitingCall(address, cw.numberType);
        }
        log("processPlusCodeForWaitingCall after format number:" + cw.number);
    }
    /// @}

    /// M: CDMA process waiting call @{
    private boolean shouldNotifyWaitingCall(CdmaCallWaitingNotification cw) {
        String address = cw.number;
        log("shouldNotifyWaitingCall, address:" + address);
        if (address != null && address.length() > 0) {
            /*GsmCdmaConnection lastFgConn
                    = (GsmCdmaConnection) (mForegroundCall.getLatestConnection());
            if (lastFgConn != null) {
                if (address.equals(lastFgConn.getAddress())) {
                    log("handleCallWaitingInfo, last acceptCall may fail, try again!");
                    try {
                        switchWaitingOrHoldingAndActive();
                    } catch (CallStateException e) {
                        Rlog.e(LOG_TAG, "Try accept waiting call fail, exception:" + e);
                    }
                    return false;
                }
            }*/
            GsmCdmaConnection lastRingConn
                    = (GsmCdmaConnection) (mRingingCall.getLatestConnection());
            if (lastRingConn != null) {
                if (address.equals(lastRingConn.getAddress())) {
                    log("handleCallWaitingInfo, skip duplicate waiting call!");
                    return false;
                }
            }
        }
        return true;
    }
    /// @}

    /// M: CDMA process call accepted @{
    private void handleCallAccepted() {
        List connections = mForegroundCall.getConnections();
        int count = connections.size();
        log("handleCallAccepted, fgcall count:" + count);
        if (count == 1) {
            GsmCdmaConnection c = (GsmCdmaConnection) connections.get(0);
            if (c.onCdmaCallAccept()) {
                mPhone.notifyCdmaCallAccepted();
            }
        }
    }
    /// @}
}
