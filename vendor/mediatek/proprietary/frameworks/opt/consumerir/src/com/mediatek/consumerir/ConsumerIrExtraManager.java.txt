/*
 * Copyright (C) 2013 The Android Open Source Project
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

package com.mediatek.consumerir;

import android.os.IBinder;
import android.os.RemoteException;
import android.os.ServiceManager;

import android.content.Context;
import android.util.Log;

import android.app.Activity;

/**
 * Class that operates consumer infrared on the device.
 * <p>
 * To obtain an instance of the system infrared transmitter, call
 * {@link android.content.Context#getSystemService(java.lang.String)
 * Context.getSystemService()} with
 * {@link android.content.Context#CONSUMER_IR_SERVICE_EX} as the argument.
 * </p>
 */
public final class ConsumerIrExtraManager {

    // Tag
    private static final String TAG = "ConsumerIrExtraManager";
    // Consumer IR Extra Service Name
    private static final String IR_EXTRA = "consumer_ir_extra";

    // Transceive Type (Transceive without HW)
    public static final int TRANSCEIVE_TYPE_GENERAL = 0;
    // Transceive Type (Transceive with HW)
    public static final int TRANSCEIVE_TYPE_IR_TX_RX = 1;

    /**
     * A callback invoked when the system successfully receive IR data.
     * @see #startReceive
     */
    public interface ReceivedCallback {

        /**
         * Called when receive the IR Patterns.
         * <p>
         * This callback is usually made on a binder thread (not the UI thread).
         * </p>
         * @param carrierFreq carrierFreq
         * @param pattern IR Patterns
         * @param size IR Patterns size
         * @see #setHotKnotMessageCallback
         */
        void onReceive(int carrierFreq, int[] pattern, int size);

        /**
         * Called when fail to receive the IR Patterns.
         * <p>
         * This callback is usually made on a binder thread (not the UI thread).
         * </p>
         * @param status
         * @see #setHotKnotMessageCallback
         */
        void onReceiveFailure(int status);
    }

    /**
     * Get Manager
     * @param context
     * @return Instance of ConsumerIrExtraActivityManager
     */
    public static synchronized ConsumerIrExtraManager getDefaultManager(Context context) {

        IBinder binder = ServiceManager.getService(IR_EXTRA);
        if (binder == null) {
            return null;
        }

        return new ConsumerIrExtraManager(context, binder);
    }

    private final IBinder mBinder;
    private final IConsumerIrExtraService mService;
    private final ConsumerIrExtraActivityManager mIrActivityManager;

    /**
     * Private constructor.
     * @hide to prevent subclassing from outside of the framework
     */
    private ConsumerIrExtraManager(Context context, IBinder binder) {

        Log.d(TAG, "ConsumerIrExtraManager()");
        mBinder = binder;
        mService = IConsumerIrExtraService.Stub.asInterface(binder);
        mIrActivityManager = new ConsumerIrExtraActivityManager(mService);
    }

    /**
     * Transceive, exchange data between apps and modules.
     * @return exchanged data
     */
    public byte[] transceive(int type, byte[] input, int offset, int len) {

        Log.d(TAG, "transceive()");
        if (mService == null || !mBinder.isBinderAlive()) {
            Log.e(TAG, "ConsumerIrExtraService die");
            return null;
        }

        // Check Type
        if (type == TRANSCEIVE_TYPE_GENERAL || type == TRANSCEIVE_TYPE_IR_TX_RX) {

        } else {
            throw new IllegalArgumentException("Unsupport transceive type");
        }

        // Check offset & length
        if (input == null) {
        } else {
            if (offset < 0 || len < 0) {
                throw new IllegalArgumentException("Offset or Length MUST NOT less than 0");
            }

            if (offset >= input.length || offset + len > input.length) {
                throw new IllegalArgumentException("Invalid Offset or Length");
            }
        }

        try {
            return mService.transceive(type, input, offset, len);
        } catch (RemoteException e) {
        }
        return null;
    }

    /**
     * startReceive, start receive IR patterns.
     * <p>
     * This method is synchronous; when it returns the pattern has been
     * transmitted. Only patterns shorter than 2 seconds will be transmitted.
     * </p>
     *
     * @param callback callback
     * @param activity activity
     * @param activities activities
     * @return
     *
     */
    public boolean startReceive(ReceivedCallback callback, Activity activity,
            Activity... activities) {

        Log.d(TAG, "startReceive()");
        if (mIrActivityManager == null) {
            Log.e(TAG, "startReceive fail , no consumer ir extra service.");
            return false;
        }

        return mIrActivityManager.startReceive(callback, activity, activities);
    }

    /**
     * stopReceive, stop receive IR patterns.
     */
    public void stopReceive() {

        Log.d(TAG, "stopReceive()");
        if (mIrActivityManager == null) {
            Log.e(TAG, "stopReceive fail , no consumer ir extra service.");
            return;
        }

        mIrActivityManager.stopReceive();
    }

    /**
     * isDefault.
     */
    public boolean isDefault() {

        if (mService == null || !mBinder.isBinderAlive()) {
            Log.e(TAG, "ConsumerIrExtraService die");
            return false;
        }

        try {
            return mService.isDefault();
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        return false;
    }

    /**
     * isInstalled.
     */
    public boolean isInstalled() {

        if (mService == null || !mBinder.isBinderAlive()) {
            Log.e(TAG, "ConsumerIrExtraService die");
            return false;
        }

        try {
            return mService.isInstalled();
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        return false;
    }

    /**
     * hasReceiver.
     * @return boolean
     */
    public boolean hasReceiver() {

        if (mService == null || !mBinder.isBinderAlive()) {
            Log.e(TAG, "ConsumerIrExtraService die");
            return false;
        }

        try {
            return mService.hasReceiver();
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        return false;
    }

    /**
     * Upload binary.
     * @param data
     * @param len
     * @return
     */
    public boolean upload(byte[] data, int len) {

        if(data == null) {
            throw new IllegalArgumentException("Data is not allowed to be NULL");
        }

        if(len > data.length) {
            len = data.length;
        }

        if (mService == null || !mBinder.isBinderAlive()) {
            Log.e(TAG, "ConsumerIrExtraService die");
            return false;
        }

        try {
            return mService.upload(data, len);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        return false;
    }

    /**
     * install.
     * @return boolean
     */
    public boolean install() {

        if (mService == null || !mBinder.isBinderAlive()) {
            Log.e(TAG, "ConsumerIrExtraService die");
            return false;
        }

        try {
            return mService.install();
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        return false;
    }

    /**
     * reset.
     */
    public void reset() {
        if (mService == null || !mBinder.isBinderAlive()) {
            Log.e(TAG, "ConsumerIrExtraService die");
            return;
        }
        try {
            mService.reset();
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    /**
     * uninstall.
     * @return boolean
     */
    public void uninstall() {
    if (mService == null || !mBinder.isBinderAlive()) {
            Log.e(TAG, "ConsumerIrExtraService die");
            return;
        }
        try {
            mService.uninstall();
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    /**
     * setDefault.
     * @return boolean
     */
    public boolean setDefault() {

        if (mService == null || !mBinder.isBinderAlive()) {
            Log.e(TAG, "ConsumerIrExtraService die");
            return false;
        }

        try {
            return mService.setDefault();
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        return false;
    }
}
