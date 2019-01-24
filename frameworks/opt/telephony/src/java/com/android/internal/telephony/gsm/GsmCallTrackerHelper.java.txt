/*
* Copyright (C) 2011-2014 MediaTek Inc.
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

package com.android.internal.telephony.gsm;

import android.os.ServiceManager;
import android.content.Context;
import android.telephony.Rlog;
import android.text.TextUtils;
import com.android.internal.telephony.PhoneConstants;
import com.android.internal.telephony.Call;
import com.android.internal.telephony.Connection;
import com.android.internal.telephony.DriverCall;
import com.android.internal.telephony.GsmCdmaCall;
import com.android.internal.telephony.GsmCdmaConnection;
import com.android.internal.telephony.GsmCdmaCallTracker;


import android.os.Bundle;
import android.os.RemoteException;

import com.android.internal.telephony.CallStateException;

/// [incoming indication]. @{
/* For adjust PhoneAPP priority, mtk04070, 20120307 */
import android.os.AsyncResult;
import android.os.Process;
import android.os.SystemProperties;
import android.telephony.PhoneNumberUtils;
/// @}

/// M: CC: Terminal Based Call Waiting @{
import static com.android.internal.telephony.TelephonyProperties.PROPERTY_TERMINAL_BASED_CALL_WAITING_MODE;
import static com.android.internal.telephony.TelephonyProperties.TERMINAL_BASED_CALL_WAITING_DISABLED;
import static com.android.internal.telephony.TelephonyProperties.TERMINAL_BASED_CALL_WAITING_ENABLED_OFF;
/// @}

import android.telephony.DisconnectCause;

/// M: For switch antenna feature @{
import com.android.internal.telephony.Phone;
import com.android.internal.telephony.PhoneFactory;
/// @}

public final class GsmCallTrackerHelper {

    static final String LOG_TAG = "GsmCallTrackerHelper";

    protected static final int EVENT_POLL_CALLS_RESULT             = 1;
    protected static final int EVENT_CALL_STATE_CHANGE             = 2;
    protected static final int EVENT_REPOLL_AFTER_DELAY            = 3;
    protected static final int EVENT_OPERATION_COMPLETE            = 4;
    protected static final int EVENT_GET_LAST_CALL_FAIL_CAUSE      = 5;

    protected static final int EVENT_SWITCH_RESULT                 = 8;
    protected static final int EVENT_RADIO_AVAILABLE               = 9;
    protected static final int EVENT_RADIO_NOT_AVAILABLE           = 10;
    protected static final int EVENT_CONFERENCE_RESULT             = 11;
    protected static final int EVENT_SEPARATE_RESULT               = 12;
    protected static final int EVENT_ECT_RESULT                    = 13;
    protected static final int EVENT_EXIT_ECM_RESPONSE_CDMA        = 14;
    protected static final int EVENT_CALL_WAITING_INFO_CDMA        = 15;
    protected static final int EVENT_THREE_WAY_DIAL_L2_RESULT_CDMA = 16;
    protected static final int EVENT_THREE_WAY_DIAL_BLANK_FLASH    = 20;

    protected static final int EVENT_MTK_BASE                      = 1000;
    /// M: CC: Proprietary incoming call handling
    protected static final int EVENT_INCOMING_CALL_INDICATION      = EVENT_MTK_BASE + 0;

    /// M: CC: Modem reset related handling
    protected static final int EVENT_RADIO_OFF_OR_NOT_AVAILABLE    = EVENT_MTK_BASE + 1;

    /// M: CC: Hangup special handling @{
    protected static final int EVENT_HANG_UP_RESULT                = EVENT_MTK_BASE + 2;
    protected static final int EVENT_DIAL_CALL_RESULT              = EVENT_MTK_BASE + 3;
    /// @}

    ///M: IMS conference call feature. @{
    protected static final int EVENT_ECONF_SRVCC_INDICATION        = EVENT_MTK_BASE + 4;
    protected static final int EVENT_ECONF_RESULT_INDICATION       = EVENT_MTK_BASE + 5;
    protected static final int EVENT_RETRIEVE_HELD_CALL_RESULT     = EVENT_MTK_BASE + 6;
    protected static final int EVENT_CALL_INFO_INDICATION          = EVENT_MTK_BASE + 7;
    /// @}

    private Context mContext;
    private GsmCdmaCallTracker mTracker;

    /// [Force release]
    private boolean hasPendingHangupRequest = false;
    private int pendingHangupRequest = 0;

    /// M: CC: Proprietary incoming call handling @{
    /// [incoming indication]
    private int pendingMTCallId = 0;
    private int pendingMTSeqNum = 0;
    /// @}

    /// M: CC: Forwarding number via EAIC @{
    // To store forwarding address from incoming call indication
    private boolean mContainForwardingAddress = false;
    private String  mForwardingAddress = null;
    private int     mForwardingAddressCallId = 0;
    /// @}

    /// M: For switch antenna feature @{
    private static final boolean MTK_SWITCH_ANTENNA_SUPPORT =
                SystemProperties.get("ro.mtk_switch_antenna").equals("1");
    /// @}

   public GsmCallTrackerHelper(Context context, GsmCdmaCallTracker tracker) {

        mContext = context;
        mTracker = tracker;
    }

    void logD(String msg) {
        Rlog.d(LOG_TAG, msg + " (slot " + mTracker.mPhone.getPhoneId() + ")");
    }

    void logI(String msg) {
        Rlog.i(LOG_TAG, msg + " (slot " + mTracker.mPhone.getPhoneId() + ")");
    }

    void logW(String msg) {
        Rlog.w(LOG_TAG, msg + " (slot " + mTracker.mPhone.getPhoneId() + ")");
    }

    public void LogerMessage(int msgType) {

        switch (msgType) {
            case EVENT_POLL_CALLS_RESULT:
                logD("handle EVENT_POLL_CALLS_RESULT");
            break;

            case EVENT_CALL_STATE_CHANGE:
                logD("handle EVENT_CALL_STATE_CHANGE");
            break;

            case EVENT_REPOLL_AFTER_DELAY:
                logD("handle EVENT_REPOLL_AFTER_DELAY");
            break;

            case EVENT_OPERATION_COMPLETE:
                logD("handle EVENT_OPERATION_COMPLETE");
            break;

            case EVENT_GET_LAST_CALL_FAIL_CAUSE:
                logD("handle EVENT_GET_LAST_CALL_FAIL_CAUSE");
            break;

            case EVENT_SWITCH_RESULT:
                logD("handle EVENT_SWITCH_RESULT");
            break;

            case EVENT_RADIO_AVAILABLE:
                logD("handle EVENT_RADIO_AVAILABLE");
            break;

            case EVENT_RADIO_NOT_AVAILABLE:
                logD("handle EVENT_RADIO_NOT_AVAILABLE");
            break;

            case EVENT_CONFERENCE_RESULT:
                logD("handle EVENT_CONFERENCE_RESULT");
            break;

            case EVENT_SEPARATE_RESULT:
                logD("handle EVENT_SEPARATE_RESULT");
            break;

            case EVENT_ECT_RESULT:
                logD("handle EVENT_ECT_RESULT");
            break;

            /* M: CC part start */
            case EVENT_HANG_UP_RESULT:
                logD("handle EVENT_HANG_UP_RESULT");
            break;

            case EVENT_DIAL_CALL_RESULT:
                logD("handle EVENT_DIAL_CALL_RESULT");
            break;

            case EVENT_INCOMING_CALL_INDICATION:
                logD("handle EVENT_INCOMING_CALL_INDICATION");
            break;

            case EVENT_RADIO_OFF_OR_NOT_AVAILABLE:
                logD("handle EVENT_RADIO_OFF_OR_NOT_AVAILABLE");
            break;
            /* M: CC part end */

            default:
                logD("handle XXXXX");
            break;

        }
    }

   public void LogState() {

        int callId = 0;
        int count = 0;

        for (int i = 0, s = mTracker.MAX_CONNECTIONS_GSM; i < s; i++) {
            if (mTracker.mConnections[i] != null) {
                callId = mTracker.mConnections[i].mIndex + 1;
                count ++;
                logI("* conn id " + callId + " existed");
            }
        }
        logI("* GsmCT has " + count + " connection");
    }

    /// M: CC: Hangup special handling @{
    public boolean ForceReleaseConnection(GsmCdmaCall call, GsmCdmaCall hangupCall) {
        GsmCdmaConnection cn;
        if (call.mState == Call.State.DISCONNECTING) {
            for (int i = 0; i < call.mConnections.size(); i++) {
                cn = (GsmCdmaConnection) call.mConnections.get(i);
                mTracker.mCi.forceReleaseCall(cn.mIndex + 1, mTracker.obtainCompleteMessage());
            }
            /* To solve ALPS01525265 */
            if (call == hangupCall) {
               return true;
            }
        }
        return false;
    }

    public boolean ForceReleaseAllConnection(GsmCdmaCall call) {
        boolean forceReleaseFg = ForceReleaseConnection(mTracker.mForegroundCall, call);
        boolean forceReleaseBg = ForceReleaseConnection(mTracker.mBackgroundCall, call);
        boolean forceReleaseRing = ForceReleaseConnection(mTracker.mRingingCall, call);

        /* To solve ALPS01525265 */
        if (forceReleaseFg || forceReleaseBg || forceReleaseRing) {
           logD("hangup(GsmCdmaCall)Hang up disconnecting call, return directly");
           return true;
        }

        return false;
    }

    // Not in use
    public boolean ForceReleaseNotRingingConnection(GsmCdmaCall call) {
        boolean forceReleaseFg = ForceReleaseConnection(mTracker.mForegroundCall, call);
        boolean forceReleaseBg = ForceReleaseConnection(mTracker.mBackgroundCall, call);

        /* To solve ALPS01525265 */
        if (forceReleaseFg || forceReleaseBg) {
           logD("hangup(GsmCdmaCall)Hang up disconnecting call, return directly");
           return true;
        }

        return true;
    }

    public boolean hangupConnectionByIndex(GsmCdmaCall c, int index) {
        int count = c.mConnections.size();
        for (int i = 0; i < count; i++) {
            GsmCdmaConnection cn = (GsmCdmaConnection) c.mConnections.get(i);
            /// M: CC: Hangup Conference special handling @{
            // Not to hangup a DISCONNECTED connection in a conference call
            // [ALPS02347071][ALPS02092424] GsmCall.mConnections NOT removed in handlePollCalls().
            if (cn.getState() == GsmCdmaCall.State.DISCONNECTED) {
                logD("hangupConnectionByIndex: hangup a DISCONNECTED conn");
                continue;
            }
            /// @}
            try {
                if (cn.getGsmCdmaIndex() == index) {
                    mTracker.mCi.hangupConnection(index, mTracker.obtainCompleteMessage());
                    return true;
                }
            } catch (CallStateException ex) {
                // Ignore "connection not found"
                // Call may have hung up already
                logW("GsmCallTracker hangupConnectionByIndex() on absent connection ");
            }
        }
        return false;
    }

    // Not in use
    public boolean hangupFgConnectionByIndex(int index) {
        return hangupConnectionByIndex(mTracker.mForegroundCall, index);
    }

    public boolean hangupBgConnectionByIndex(int index) {
        return hangupConnectionByIndex(mTracker.mBackgroundCall, index);
    }

    public boolean hangupRingingConnectionByIndex(int index) {
        return hangupConnectionByIndex(mTracker.mRingingCall, index);
    }


    public int getCurrentTotalConnections() {
        int count = 0;
        for (int i = 0; i < mTracker.MAX_CONNECTIONS_GSM; i++) {
            if (mTracker.mConnections[i] != null) {
                count ++;
            }
            }
        return count;
    }

    public void PendingHangupRequestUpdate() {
        logD("updatePendingHangupRequest - " + mTracker.mHangupPendingMO +
                hasPendingHangupRequest + pendingHangupRequest);
        if (mTracker.mHangupPendingMO) {
            if (hasPendingHangupRequest) {
                pendingHangupRequest--;
                if (pendingHangupRequest == 0) {
                    hasPendingHangupRequest = false;
                }
            }
        }
    }

    public void PendingHangupRequestInc() {
        hasPendingHangupRequest = true;
        pendingHangupRequest++;
    }

    public void PendingHangupRequestDec() {
        if (hasPendingHangupRequest) {
            pendingHangupRequest--;
            if (pendingHangupRequest == 0) {
                hasPendingHangupRequest = false;
            }
        }
    }

    public void PendingHangupRequestReset() {
        hasPendingHangupRequest = false;
        pendingHangupRequest = 0;
    }

    public boolean hasPendingHangupRequest() {
        return hasPendingHangupRequest;
    }
    /// @}

    /// M: CC: Proprietary incoming call handling @{
    // Not in use
    public int CallIndicationGetId() {
        return pendingMTCallId;
    }

    // Not in use
    public int CallIndicationGetSeqNo() {
        return pendingMTSeqNum;
    }

    public void CallIndicationProcess(AsyncResult ar) {
        int mode = 0;
        String[] incomingCallInfo = (String[]) ar.result;
        int callId = Integer.parseInt(incomingCallInfo[0]);
        int callMode = Integer.parseInt(incomingCallInfo[3]);
        int seqNumber = Integer.parseInt(incomingCallInfo[4]);

        logD("CallIndicationProcess " + mode + " pendingMTCallId " + pendingMTCallId +
                " pendingMTSeqNum " + pendingMTSeqNum);

        /// M: CC: Terminal Based Call Waiting @{
        String tbcwMode = mTracker.mPhone.getSystemProperty(
                PROPERTY_TERMINAL_BASED_CALL_WAITING_MODE,
                TERMINAL_BASED_CALL_WAITING_DISABLED);
        GsmCdmaCall.State fgState = (mTracker.mForegroundCall == null) ? GsmCdmaCall.State.IDLE
                : mTracker.mForegroundCall.getState();
        GsmCdmaCall.State bgState = (mTracker.mBackgroundCall == null) ? GsmCdmaCall.State.IDLE
                : mTracker.mBackgroundCall.getState();

        logD("PROPERTY_TERMINAL_BASED_CALL_WAITING_MODE = " + tbcwMode
                + " , ForgroundCall State = " + fgState
                + " , BackgroundCall State = " + bgState);
        if (TERMINAL_BASED_CALL_WAITING_ENABLED_OFF.equals(tbcwMode)
                && ((fgState == GsmCdmaCall.State.ACTIVE) ||
                (bgState == GsmCdmaCall.State.HOLDING))) {
            logD("PROPERTY_TERMINAL_BASED_CALL_WAITING_MODE = "
                    + "TERMINAL_BASED_CALL_WAITING_ENABLED_OFF."
                    + " Reject the call as UDUB ");
            mode = 1; //1:disallow MT call
            mTracker.mCi.setCallIndication(mode, callId, seqNumber, null);
            //Terminal Based Call Waiting OFF:Silent Reject without generating missed call log
            return;
        }
        /// @}

        /// M: CC: Forwarding number via EAIC @{
        /* Check if EAIC message contains forwarding address(A calls B and it is forwarded to C,
             C may receive forwarding address - B's phone number). */
        mForwardingAddress = null;
        if ((incomingCallInfo[5] != null) && (incomingCallInfo[5].length() > 0)) {
            /* This value should be set to true after CallManager approves the incoming call */
            mContainForwardingAddress = false;
            mForwardingAddress = incomingCallInfo[5];
            mForwardingAddressCallId = callId;
            logD("EAIC message contains forwarding address - " + mForwardingAddress + "," + callId);
        }
        /// @}

        /// M: CC: Proprietary incoming call handling @{
        /// Reject MT when another MT already exists via EAIC disapproval
        if (mTracker.mState == PhoneConstants.State.RINGING) {
            mode = 1;
        }
        /// @}
        /// M: For 3G VT only @{
        else if (mTracker.mState == PhoneConstants.State.OFFHOOK) {
            if (callMode == 10) {
                // incoming VT call, reject it since one active call (VT or voice) already exists
                // FIXME: will a new VT call comes if one VT call already exists?
                for (int i = 0; i < mTracker.MAX_CONNECTIONS_GSM; i++) {
                    Connection cn = mTracker.mConnections[i];
                    if (cn != null) {
                        mode = 1;
                        break;
                    }
                }
            } else if (callMode == 0) {
                // incoming voice call, reject new voice call since one VT call already exists
                for (int i = 0; i < mTracker.MAX_CONNECTIONS_GSM; i++) {
                    Connection cn = mTracker.mConnections[i];
                    if (cn != null && cn.isVideo()) {
                        mode = 1;
                        break;
                    }
                }
            } else {
                // the incoming call is neither VT nor voice call, reject it anyway.
                mode = 1;
            }
        }
        /// @}

        /* To raise PhoneAPP priority to avoid delaying incoming call screen to be showed,
             mtk04070, 20120307 */
        if (mode == 0) {
            pendingMTCallId = callId;
            pendingMTSeqNum = seqNumber;
            mTracker.mVoiceCallIncomingIndicationRegistrants.notifyRegistrants();
            logD("notify mVoiceCallIncomingIndicationRegistrants " + pendingMTCallId + " " +
                    pendingMTSeqNum);
        }

        /// M: CC: Proprietary incoming call handling @{
        /// Reject MT when another MT already exists via EAIC disapproval
        if (mode == 1) {
            DriverCall dc = new DriverCall();
            dc.isMT = true;
            dc.index = callId;
            dc.state = DriverCall.State.WAITING;

            mTracker.mCi.setCallIndication(mode, callId, seqNumber, null);

            /// M: For 3G VT only @{
            //dc.isVoice = true;
            if (callMode == 0) {
                dc.isVoice = true;
                dc.isVideo = false;
            } else if (callMode == 10) {
                dc.isVoice = false;
                dc.isVideo = true;
            } else {
                dc.isVoice = false;
                dc.isVideo = false;
            }
            /// @}
            dc.number = incomingCallInfo[1];
            dc.numberPresentation = PhoneConstants.PRESENTATION_ALLOWED;
            dc.TOA = Integer.parseInt(incomingCallInfo[2]);
            dc.number = PhoneNumberUtils.stringFromStringAndTOA(dc.number, dc.TOA);

            GsmCdmaConnection cn = new GsmCdmaConnection(mTracker.mPhone, dc, mTracker, callId);
            cn.onReplaceDisconnect(DisconnectCause.INCOMING_MISSED);
        }
        /// @}
    }

    public void CallIndicationResponse(boolean accept) {
        int mode = 0;

        logD("setIncomingCallIndicationResponse " + mode + " pendingMTCallId " + pendingMTCallId +
                " pendingMTSeqNum " + pendingMTSeqNum);

        if (accept) {
            int pid = Process.myPid();

            mode = 0;
            Process.setThreadPriority(pid, Process.THREAD_PRIORITY_DEFAULT - 10);
            logD("Adjust the priority of process - " + pid + " to " +
                    Process.getThreadPriority(pid));

            /// M: CC: Forwarding number via EAIC @{
            /* EAIC message contains forwarding address(A calls B and it is forwarded to C,
                 C may receive forwarding number - B's phone number). */
            if (mForwardingAddress != null) {
                mContainForwardingAddress = true;
            }
            /// @}

        } else {
            mode = 1;
        }
        mTracker.mCi.setCallIndication(mode, pendingMTCallId, pendingMTSeqNum, null);
        pendingMTCallId = 0;
        pendingMTSeqNum = 0;
    }

    public void CallIndicationEnd() {

        /* To adjust PhoneAPP priority to normal, mtk04070, 20120307 */
        int pid = Process.myPid();
        if (Process.getThreadPriority(pid) != Process.THREAD_PRIORITY_DEFAULT) {
            Process.setThreadPriority(pid, Process.THREAD_PRIORITY_DEFAULT);
            logD("Current priority = " + Process.getThreadPriority(pid));
        }
    }
    /// @}

    /// M: CC: Forwarding number via EAIC @{
    /**
      *  To clear forwarding address variables
      */
    public void clearForwardingAddressVariables(int index) {
       if (mContainForwardingAddress &&
       (mForwardingAddressCallId == (index + 1))) {
          mContainForwardingAddress = false;
          mForwardingAddress = null;
          mForwardingAddressCallId = 0;
       }
    }

    /**
      *  To clear forwarding address variables
      */
    public void setForwardingAddressToConnection(int index, Connection conn) {
        if (mContainForwardingAddress &&
             (mForwardingAddress != null) &&
             (mForwardingAddressCallId == (index + 1))) {
             conn.setForwardingAddress(mForwardingAddress);
             logD("Store forwarding address - " + mForwardingAddress);
             logD("Get forwarding address - " + conn.getForwardingAddress());
             clearForwardingAddressVariables(index);
        }
    }
    /// @}

    private boolean shouldNotifyMtCall() {
        if (MTK_SWITCH_ANTENNA_SUPPORT) {
            logD("shouldNotifyMtCall, mTracker.mPhone:" + mTracker.mPhone);
            Phone[] phones = PhoneFactory.getPhones();
            for (Phone phone : phones) {
                logD("phone:" + phone + ", state:" + phone.getState());
                if (phone.getState() != PhoneConstants.State.IDLE && phone != mTracker.mPhone) {
                    logD("shouldNotifyMtCall, another phone active, phone:" + phone
                            + ", state:" + phone.getState());
                    return false;
                }
            }
        }
        return true;
    }
}

