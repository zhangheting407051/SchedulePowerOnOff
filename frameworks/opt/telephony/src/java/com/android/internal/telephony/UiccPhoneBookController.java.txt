/*
* Copyright (C) 2014 MediaTek Inc.
* Modification based on code covered by the mentioned copyright
* and/or permission notice(s).
*/
/*
 * Copyright (C) 2008 The Android Open Source Project
 * Copyright (c) 2011-2013, The Linux Foundation. All rights reserved.
 * Not a Contribution.
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

import android.os.ServiceManager;
import android.os.RemoteException;
import android.telephony.Rlog;

import com.android.internal.telephony.IIccPhoneBook;
import com.android.internal.telephony.uicc.AdnRecord;

import com.mediatek.internal.telephony.uicc.AlphaTag;
import com.mediatek.internal.telephony.uicc.UsimGroup;
import com.mediatek.internal.telephony.uicc.UsimPBMemInfo;
import java.lang.ArrayIndexOutOfBoundsException;
import java.lang.NullPointerException;
import java.util.List;

public class UiccPhoneBookController extends IIccPhoneBook.Stub {
    private static final String TAG = "UiccPhoneBookController";
    private Phone[] mPhone;

    /* only one UiccPhoneBookController exists */
    public UiccPhoneBookController(Phone[] phone) {
        if (ServiceManager.getService("simphonebook") == null) {
               ServiceManager.addService("simphonebook", this);
        }
        mPhone = phone;
    }

    @Override
    public boolean
    updateAdnRecordsInEfBySearch (int efid, String oldTag, String oldPhoneNumber,
            String newTag, String newPhoneNumber, String pin2) throws android.os.RemoteException {
        return updateAdnRecordsInEfBySearchForSubscriber(getDefaultSubscription(), efid, oldTag,
                oldPhoneNumber, newTag, newPhoneNumber, pin2);
    }

    @Override
    public boolean
    updateAdnRecordsInEfBySearchForSubscriber(int subId, int efid, String oldTag,
            String oldPhoneNumber, String newTag, String newPhoneNumber,
            String pin2) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.updateAdnRecordsInEfBySearch(efid, oldTag,
                    oldPhoneNumber, newTag, newPhoneNumber, pin2);
        } else {
            Rlog.e(TAG,"updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:"+subId);
            return false;
        }
    }

    public int
    updateAdnRecordsInEfBySearchWithError(int subId, int efid,
            String oldTag, String oldPhoneNumber,
            String newTag, String newPhoneNumber, String pin2) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.updateAdnRecordsInEfBySearchWithError(efid,
                    oldTag, oldPhoneNumber, newTag, newPhoneNumber, pin2);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return 0;
        }
    }

    public int
    updateUsimPBRecordsInEfBySearchWithError(int subId, int efid,
            String oldTag, String oldPhoneNumber, String oldAnr, String oldGrpIds,
            String[] oldEmails, String newTag, String newPhoneNumber, String newAnr,
            String newGrpIds, String[] newEmails) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.updateUsimPBRecordsInEfBySearchWithError(
                    efid, oldTag, oldPhoneNumber, oldAnr, oldGrpIds, oldEmails,
                    newTag, newPhoneNumber, newAnr, newGrpIds, newEmails);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return 0;
        }
    }
    @Override
    public boolean
    updateAdnRecordsInEfByIndex(int efid, String newTag,
            String newPhoneNumber, int index, String pin2) throws android.os.RemoteException {
        return updateAdnRecordsInEfByIndexForSubscriber(getDefaultSubscription(), efid, newTag,
                newPhoneNumber, index, pin2);
    }

    @Override
    public boolean
    updateAdnRecordsInEfByIndexForSubscriber(int subId, int efid, String newTag,
            String newPhoneNumber, int index, String pin2) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.updateAdnRecordsInEfByIndex(efid, newTag,
                    newPhoneNumber, index, pin2);
        } else {
            Rlog.e(TAG,"updateAdnRecordsInEfByIndex iccPbkIntMgr is" +
                      " null for Subscription:"+subId);
            return false;
        }
    }

    public int
    updateAdnRecordsInEfByIndexWithError(int subId, int efid, String newTag,
            String newPhoneNumber, int index, String pin2) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.updateAdnRecordsInEfByIndexWithError(efid,
                    newTag, newPhoneNumber, index, pin2);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return 0;
        }
    }

    public int
    updateUsimPBRecordsInEfByIndexWithError(int subId, int efid, String newTag,
            String newPhoneNumber, String newAnr,  String newGrpIds, String[] newEmails, int index)
            throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.updateUsimPBRecordsInEfByIndexWithError(efid,
                    newTag, newPhoneNumber, newAnr, newGrpIds, newEmails, index);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return 0;
        }
    }

    public int updateUsimPBRecordsByIndexWithError(int subId, int efid, AdnRecord record, int index)
            throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.updateUsimPBRecordsByIndexWithError(efid,
                    record, index);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return 0;
        }
    }

    public int updateUsimPBRecordsBySearchWithError(int subId, int efid, AdnRecord oldAdn,
            AdnRecord newAdn) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.updateUsimPBRecordsBySearchWithError(efid,
                    oldAdn, newAdn);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return 0;
        }
    }
    @Override
    public int[] getAdnRecordsSize(int efid) throws android.os.RemoteException {
        return getAdnRecordsSizeForSubscriber(getDefaultSubscription(), efid);
    }

    @Override
    public int[]
    getAdnRecordsSizeForSubscriber(int subId, int efid) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.getAdnRecordsSize(efid);
        } else {
            Rlog.e(TAG,"getAdnRecordsSize iccPbkIntMgr is" +
                      " null for Subscription:"+subId);
            return null;
        }
    }

    @Override
    public List<AdnRecord> getAdnRecordsInEf(int efid) throws android.os.RemoteException {
        return getAdnRecordsInEfForSubscriber(getDefaultSubscription(), efid);
    }

    @Override
    public List<AdnRecord> getAdnRecordsInEfForSubscriber(int subId, int efid)
           throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.getAdnRecordsInEf(efid);
        } else {
            Rlog.e(TAG,"getAdnRecordsInEf iccPbkIntMgr is" +
                      "null for Subscription:"+subId);
            return null;
        }
    }

    public boolean isPhbReady(int subId) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.isPhbReady();
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return false;
        }
    }

    public List<UsimGroup> getUsimGroups(int subId) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.getUsimGroups();
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return null;
        }
    }

    public String getUsimGroupById(int subId, int nGasId) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.getUsimGroupById(nGasId);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return null;
        }
    }

    public boolean removeUsimGroupById(int subId, int nGasId) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.removeUsimGroupById(nGasId);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return false;
        }
    }

    public int insertUsimGroup(int subId, String grpName) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.insertUsimGroup(grpName);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return -1;
        }
    }

    public int updateUsimGroup(int subId, int nGasId, String grpName)
            throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.updateUsimGroup(nGasId, grpName);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return -1;
        }
    }

    public boolean addContactToGroup(int subId, int adnIndex, int grpIndex)
            throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.addContactToGroup(adnIndex, grpIndex);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return false;
        }
    }

    public boolean removeContactFromGroup(int subId, int adnIndex, int grpIndex)
            throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.removeContactFromGroup(adnIndex, grpIndex);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return false;
        }
    }

    public boolean updateContactToGroups(int subId, int adnIndex, int[] grpIdList)
            throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.updateContactToGroups(adnIndex, grpIdList);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return false;
        }
    }

    public boolean moveContactFromGroupsToGroups(int subId, int adnIndex, int[] fromGrpIdList,
                                             int[] toGrpIdList) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.moveContactFromGroupsToGroups(adnIndex,
                    fromGrpIdList, toGrpIdList);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return false;
        }
    }

    public int hasExistGroup(int subId, String grpName) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.hasExistGroup(grpName);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return -1;
        }
    }

    public int getUsimGrpMaxNameLen(int subId) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.getUsimGrpMaxNameLen();
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return -1;
        }
    }

    public int getUsimGrpMaxCount(int subId) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.getUsimGrpMaxCount();
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return -1;
        }
    }

    public List<AlphaTag> getUsimAasList(int subId) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.getUsimAasList();
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return null;
        }
    }

    public String getUsimAasById(int subId, int index) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.getUsimAasById(index);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return null;
        }
    }

    public int insertUsimAas(int subId, String aasName) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.insertUsimAas(aasName);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return 0;
        }
    }

    public int getAnrCount(int subId) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.getAnrCount();
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return 0;
        }
    }

    public int getEmailCount(int subId) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.getEmailCount();
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return 0;
        }
    }

    public int getUsimAasMaxCount(int subId) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.getUsimAasMaxCount();
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return 0;
        }
    }

    public int getUsimAasMaxNameLen(int subId) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.getUsimAasMaxNameLen();
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return 0;
        }
    }

    public boolean updateUsimAas(int subId, int index, int pbrIndex, String aasName)
            throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.updateUsimAas(index, pbrIndex, aasName);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return false;
        }
    }

    public boolean removeUsimAasById(int subId, int index, int pbrIndex)
            throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.removeUsimAasById(index, pbrIndex);
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return false;
        }
    }

    public boolean hasSne(int subId) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.hasSne();
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return false;
        }
    }

    public int getSneRecordLen(int subId) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.getSneRecordLen();
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return 0;
        }
    }

    public boolean isAdnAccessible(int subId) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.isAdnAccessible();
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return true;
        }
    }

    public UsimPBMemInfo[] getPhonebookMemStorageExt(int subId) throws android.os.RemoteException {
        IccPhoneBookInterfaceManager iccPbkIntMgr =
                             getIccPhoneBookInterfaceManager(subId);
        if (iccPbkIntMgr != null) {
            return iccPbkIntMgr.getPhonebookMemStorageExt();
        } else {
            Rlog.e(TAG, "updateAdnRecordsInEfBySearch iccPbkIntMgr is" +
                      " null for Subscription:" + subId);
            return null;
        }
    }
    /**
     * get phone book interface manager object based on subscription.
     **/
    private IccPhoneBookInterfaceManager
            getIccPhoneBookInterfaceManager(int subId) {

        int phoneId = SubscriptionController.getInstance().getPhoneId(subId);
        try {
            return mPhone[phoneId].getIccPhoneBookInterfaceManager();
        } catch (NullPointerException e) {
            Rlog.e(TAG, "Exception is :"+e.toString()+" For subscription :"+subId );
            e.printStackTrace(); //To print stack trace
            return null;
        } catch (ArrayIndexOutOfBoundsException e) {
            Rlog.e(TAG, "Exception is :"+e.toString()+" For subscription :"+subId );
            e.printStackTrace();
            return null;
        }
    }

    private int getDefaultSubscription() {
        return PhoneFactory.getDefaultSubscription();
    }
}
