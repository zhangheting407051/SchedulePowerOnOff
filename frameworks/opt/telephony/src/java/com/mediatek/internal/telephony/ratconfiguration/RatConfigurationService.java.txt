/* Copyright Statement:
 *
 * This software/firmware and related documentation ("MediaTek Software") are
 * protected under relevant copyright laws. The information contained herein
 * is confidential and proprietary to MediaTek Inc. and/or its licensors.
 * Without the prior written permission of MediaTek inc. and/or its licensors,
 * any reproduction, modification, use or disclosure of MediaTek Software,
 * and information contained herein, in whole or in part, shall be strictly prohibited.
 */
/* MediaTek Inc. (C) 2015. All rights reserved.
 *
 * BY OPENING THIS FILE, RECEIVER HEREBY UNEQUIVOCALLY ACKNOWLEDGES AND AGREES
 * THAT THE SOFTWARE/FIRMWARE AND ITS DOCUMENTATIONS ("MEDIATEK SOFTWARE")
 * RECEIVED FROM MEDIATEK AND/OR ITS REPRESENTATIVES ARE PROVIDED TO RECEIVER ON
 * AN "AS-IS" BASIS ONLY. MEDIATEK EXPRESSLY DISCLAIMS ANY AND ALL WARRANTIES,
 * EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF
 * MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE OR NONINFRINGEMENT.
 * NEITHER DOES MEDIATEK PROVIDE ANY WARRANTY WHATSOEVER WITH RESPECT TO THE
 * SOFTWARE OF ANY THIRD PARTY WHICH MAY BE USED BY, INCORPORATED IN, OR
 * SUPPLIED WITH THE MEDIATEK SOFTWARE, AND RECEIVER AGREES TO LOOK ONLY TO SUCH
 * THIRD PARTY FOR ANY WARRANTY CLAIM RELATING THERETO. RECEIVER EXPRESSLY ACKNOWLEDGES
 * THAT IT IS RECEIVER'S SOLE RESPONSIBILITY TO OBTAIN FROM ANY THIRD PARTY ALL PROPER LICENSES
 * CONTAINED IN MEDIATEK SOFTWARE. MEDIATEK SHALL ALSO NOT BE RESPONSIBLE FOR ANY MEDIATEK
 * SOFTWARE RELEASES MADE TO RECEIVER'S SPECIFICATION OR TO CONFORM TO A PARTICULAR
 * STANDARD OR OPEN FORUM. RECEIVER'S SOLE AND EXCLUSIVE REMEDY AND MEDIATEK'S ENTIRE AND
 * CUMULATIVE LIABILITY WITH RESPECT TO THE MEDIATEK SOFTWARE RELEASED HEREUNDER WILL BE,
 * AT MEDIATEK'S OPTION, TO REVISE OR REPLACE THE MEDIATEK SOFTWARE AT ISSUE,
 * OR REFUND ANY SOFTWARE LICENSE FEES OR SERVICE CHARGE PAID BY RECEIVER TO
 * MEDIATEK FOR SUCH MEDIATEK SOFTWARE AT ISSUE.
 *
 * The following software/firmware and/or related documentation ("MediaTek Software")
 * have been modified by MediaTek Inc. All revisions are subject to any receiver's
 * applicable license agreements with MediaTek Inc.
 */


package com.mediatek.internal.telephony.ratconfiguration;

import com.mediatek.internal.telephony.ratconfiguration.RatConfiguration;
import android.provider.Settings;
import android.telephony.Rlog;
import android.telephony.SubscriptionManager;
import android.telephony.TelephonyManager;
import com.android.internal.telephony.CommandsInterface;
import com.android.internal.telephony.Phone;
import com.android.internal.telephony.PhoneFactory;
import com.android.internal.telephony.ProxyController;
import com.android.internal.telephony.PhoneConstants;
import com.android.internal.telephony.RILConstants;
import com.android.internal.telephony.SubscriptionController;
import android.os.SystemProperties;


public class RatConfigurationService {
    private static final String LOG_TAG = "RatConfigurationService";
    private volatile static RatConfigurationService instance = null;

    private static final int MD_MODE_RAT_CONFIG_OFFSET   = 8;
    private static final String PROPERTY_MAJOR_SIM = "persist.radio.simswitch";
    private static final int MAJOR_SIM_UNKNOWN    = -99;

    private static final int PROJECT_SIM_NUM = TelephonyManager.getDefault().getSimCount();
    private static Phone[] sProxyPhones = null;
    private static Phone[] sActivePhones = new Phone[PROJECT_SIM_NUM];
    private static CommandsInterface[] sCi = new CommandsInterface[PROJECT_SIM_NUM];

    static final String PROPERTY_PS2_RAT_CONFIG = "persist.radio.mtk_ps2_rat";

    private RatConfigurationService() {
      sProxyPhones = PhoneFactory.getPhones();
      for (int i = 0; i < PROJECT_SIM_NUM; i++) {
        sActivePhones[i] = sProxyPhones[i];
        sCi[i] = sActivePhones[i].mCi;
      }
    };

    public static RatConfigurationService getInstance() {
      if (instance == null) {
        synchronized(RatConfigurationService.class) {
          if (instance == null) {
            instance = new RatConfigurationService();
          }
        }
      }
      return instance;
    }

    /*
     * get current active rat.
     * @return Stirng format like C/Lf/Lt/W/T/G
     */
    public String getActiveRatConfig() {
        String result = RatConfiguration.getActiveRatConfig();
        logd("getActiveRatConfig " + result);
        return result;
    }
    private void resetPs2Rat(int iRat) {
        //For PS1 C2K 4M proj, C/Lf/Lt/G, PS2 need to remove the support of W
        if (iRat == (RatConfiguration.MASK_CDMA
                | RatConfiguration.MASK_LteFdd
                | RatConfiguration.MASK_LteTdd
                | RatConfiguration.MASK_GSM)) {
            String ps2Rat = SystemProperties.get(PROPERTY_PS2_RAT_CONFIG,"");
            if (ps2Rat.contains("L") && ps2Rat.contains("G")) {
                SystemProperties.set(PROPERTY_PS2_RAT_CONFIG,"L/G");
            } else if (ps2Rat.contains("G")) {
                SystemProperties.set(PROPERTY_PS2_RAT_CONFIG,"G");
            }
        }
    }
    private boolean onRatConfigChanged(int iRat, boolean same_rat) {
        resetNetworkSettingsByRatConfig(iRat);
        if (!same_rat) {
            resetPs2Rat(iRat);
        }
        return true;
    }
    /*
     * set current active rat.
     * @param String rat format like C/Lf/Lt/W/T/G
     * @return RatConfiguration.Result as following
     *     0 as RatConfiguration.Result.FAIL :
               fail
     *     1 as RatConfiguration.Result.SUCCESS_REBOOT:
               success, and reboot is necessary
     *     2 as RatConfiguration.Result.SUCCESS_NO_REBOOT:
               success, no need to reboot
     */
    public int setRatConfig(String rat) {
        int iRat = RatConfiguration.ratToBitmask(rat) ;
        logd("setRatConfig "+iRat);
        if (RatConfiguration.checkRatConfig(iRat) == false) {
            return 0;
        }
        if (iRat == RatConfiguration.getRatConfig()) {
            onRatConfigChanged(iRat, true);
            logd("already have these rat " + rat);
            logd("setRatConfig SUCCESS_NO_REBOOT");
            return 2;
        }
        if (setRat(iRat) == true) {
            onRatConfigChanged(iRat, false);
            RatConfiguration.updateRatConfig(iRat);
            logd("setRatConfig SUCCESS_REBOOT");
            return 1;
        } else {
            logd("setRat FAIL");
            return 0;
        }
    }

    private void resetNetworkSettingsByRatConfig(int iRat) {
        boolean isLteTddSupported = (iRat& RatConfiguration.MASK_LteTdd) ==
                RatConfiguration.MASK_LteTdd ? true : false;
        boolean isLteFddSupported = (iRat& RatConfiguration.MASK_LteFdd) ==
                RatConfiguration.MASK_LteFdd ? true : false;
        boolean isLteSupported = isLteFddSupported ||isLteTddSupported;
        boolean isC2kSupported = (iRat& RatConfiguration.MASK_CDMA) ==
                RatConfiguration.MASK_CDMA ? true : false;
        int networkMode = 0;

        logd("[resetNetworkSettingsByRatConfig]+");

        if (isC2kSupported) {
            if (isLteSupported) {
                networkMode = RILConstants.NETWORK_MODE_LTE_CDMA_EVDO_GSM_WCDMA;
            } else {
                networkMode = RILConstants.NETWORK_MODE_GLOBAL;
            }
        } else {
            if (isLteSupported) {
                networkMode = RILConstants.NETWORK_MODE_LTE_GSM_WCDMA;
            } else {
                networkMode = RILConstants.NETWORK_MODE_WCDMA_PREF;
            }
        }

        for (int i = 0; i < PROJECT_SIM_NUM; i++) {
            int phoneSubId = SubscriptionController.getInstance().getSubIdUsingPhoneId(i);
            logd("[resetNetworkSettingsByRatConfig] phoneId:" +
                    i + ", subId:" + phoneSubId + ", nwMode:" + networkMode);
            if (SubscriptionManager.isValidSubscriptionId(phoneSubId)) {
                Settings.Global.putInt(sActivePhones[i].getContext().getContentResolver(),
                        Settings.Global.PREFERRED_NETWORK_MODE + phoneSubId, networkMode);
                logd("[resetNetworkSettingsByRatConfig]update Settings successfully");
            } else {
                logd("[resetNetworkSettingsByRatConfig]SubId is invalid");
            }
        }
        logd("[resetNetworkSettingsByRatConfig]-");
    }

    private int ratToModemMode(int iRat) {
               int modemMode = RatConfiguration.MD_MODE_UNKNOWN;
               if (iRat == (RatConfiguration.MASK_LteFdd |
                           RatConfiguration.MASK_LteTdd |
                           RatConfiguration.MASK_TDSCDMA |
                           RatConfiguration.MASK_GSM)) {
                   modemMode = RatConfiguration.MD_MODE_LTG;
               } else if (iRat == (RatConfiguration.MASK_LteTdd |
                           RatConfiguration.MASK_TDSCDMA |
                           RatConfiguration.MASK_GSM)) {
                   modemMode = RatConfiguration.MD_MODE_LTTG;
               } else if (iRat == (RatConfiguration.MASK_LteFdd |
                           RatConfiguration.MASK_LteTdd |
                           RatConfiguration.MASK_TDSCDMA |
                           RatConfiguration.MASK_WCDMA |
                           RatConfiguration.MASK_GSM)) {
                   modemMode = RatConfiguration.MD_MODE_LWTG;
               } else if (iRat == (RatConfiguration.MASK_LteFdd |
                           RatConfiguration.MASK_WCDMA |
                           RatConfiguration.MASK_GSM)) {
                   modemMode = RatConfiguration.MD_MODE_LFWG;
               } else if (iRat == (RatConfiguration.MASK_LteFdd |
                           RatConfiguration.MASK_LteTdd |
                           RatConfiguration.MASK_WCDMA |
                           RatConfiguration.MASK_GSM)) {
                   modemMode = RatConfiguration.MD_MODE_LWG;
               } else if (iRat == (RatConfiguration.MASK_CDMA |
                           RatConfiguration.MASK_LteFdd |
                           RatConfiguration.MASK_LteTdd |
                           RatConfiguration.MASK_TDSCDMA |
                           RatConfiguration.MASK_WCDMA |
                           RatConfiguration.MASK_GSM)) {
                   modemMode = RatConfiguration.MD_MODE_LWCTG;
               }  else if (iRat == (RatConfiguration.MASK_CDMA |
                           RatConfiguration.MASK_LteFdd |
                           RatConfiguration.MASK_LteTdd |
                           RatConfiguration.MASK_TDSCDMA |
                           RatConfiguration.MASK_GSM)) {
                   modemMode = RatConfiguration.MD_MODE_LCTG;
               }  else if (iRat == (RatConfiguration.MASK_CDMA |
                           RatConfiguration.MASK_LteTdd |
                           RatConfiguration.MASK_TDSCDMA |
                           RatConfiguration.MASK_GSM)) {
                   modemMode = RatConfiguration.MD_MODE_LTCTG;
               } else if (iRat == (RatConfiguration.MASK_CDMA |
                           RatConfiguration.MASK_LteFdd |
                           RatConfiguration.MASK_WCDMA |
                           RatConfiguration.MASK_GSM)) {
                   modemMode = RatConfiguration.MD_MODE_LFWCG;
               } else if (iRat == (RatConfiguration.MASK_CDMA |
                           RatConfiguration.MASK_LteFdd |
                           RatConfiguration.MASK_LteTdd |
                           RatConfiguration.MASK_WCDMA |
                           RatConfiguration.MASK_GSM)) {
                   modemMode = RatConfiguration.MD_MODE_LWCG;
             } else if (iRat == (RatConfiguration.MASK_CDMA |
                           RatConfiguration.MASK_WCDMA |
                           RatConfiguration.MASK_GSM)) {
                   modemMode = RatConfiguration.MD_MODE_LWCG;
             } else if (iRat == (RatConfiguration.MASK_CDMA |
                           RatConfiguration.MASK_LteFdd |
                           RatConfiguration.MASK_LteTdd |
                           RatConfiguration.MASK_GSM)) {
                   modemMode = RatConfiguration.MD_MODE_LWCG;
             }
             logd("ratToModemMode: " + modemMode);
             return modemMode;
       }

    private boolean setRat(int iRat) {
        boolean result;
        int modemMode = ratToModemMode(iRat);
        int protocolSim = getMajorSim();
        logd("[ReloadStoreTRM]protocolSim: " + protocolSim);
        if (protocolSim >= PhoneConstants.SIM_ID_1 &&
            protocolSim <= PhoneConstants.SIM_ID_4) {
            result = setMode(sCi[protocolSim], (modemMode & (0xff) )|
                    ((iRat & (0xff))<< MD_MODE_RAT_CONFIG_OFFSET) );
        } else {
            result = setMode(sCi[PhoneConstants.SIM_ID_1],(modemMode & (0xff) )|
                    ((iRat & (0xff))<< MD_MODE_RAT_CONFIG_OFFSET) );
        }
        boolean isCdmaSupport =
                (iRat & RatConfiguration.MASK_CDMA) ==
                RatConfiguration.MASK_CDMA ? true : false;
        boolean isLteFddSupport =
                (iRat& RatConfiguration.MASK_LteFdd) ==
                RatConfiguration.MASK_LteFdd ? true : false;
        boolean isLteTddSupport =
                (iRat& RatConfiguration.MASK_LteTdd) ==
                RatConfiguration.MASK_LteTdd ? true : false;
        boolean isWcdmaSupport =
                (iRat& RatConfiguration.MASK_WCDMA) ==
                RatConfiguration.MASK_WCDMA ? true : false;
        boolean isTdscdmaSupport =
                (iRat& RatConfiguration.MASK_TDSCDMA) ==
                RatConfiguration.MASK_TDSCDMA ? true : false;
        boolean isGsmSupport =
                (iRat& RatConfiguration.MASK_GSM) ==
                RatConfiguration.MASK_GSM ? true : false;
        logd("setRat -modem="+modemMode+" rat="+iRat);
        logd("setRat -"+
                isCdmaSupport+"/"+isLteFddSupport+"/"+isLteTddSupport+"/"+
                isWcdmaSupport+"/"+isTdscdmaSupport+"/"+isGsmSupport);
        return result;
    }

    private static boolean setMode(CommandsInterface ci, int modemMode) {
        if (ci.getRadioState() ==
                CommandsInterface.RadioState.RADIO_UNAVAILABLE) {
            logd("[setMode]Radio unavailable, can not switch");
            return false;
        }
        ci.storeModemType(modemMode, null);
        return true;
    }

    private static int getMajorSim() {
        String currMajorSim = SystemProperties.get(PROPERTY_MAJOR_SIM, "");
        if (currMajorSim != null && !currMajorSim.equals("")) {
            logd("[getMajorSim]: " + ((Integer.parseInt(currMajorSim)) - 1));
            return (Integer.parseInt(currMajorSim)) - 1;
        } else {
            logd("[getMajorSim]: fail to get major SIM");
            return MAJOR_SIM_UNKNOWN;
        }
    }

    private static void logd(String msg) {
        Rlog.d(LOG_TAG, msg);
    }
}

