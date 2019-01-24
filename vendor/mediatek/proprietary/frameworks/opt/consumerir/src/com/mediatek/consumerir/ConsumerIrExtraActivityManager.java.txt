package com.mediatek.consumerir;

import android.app.Activity;
import android.app.Application;
import android.app.Application.ActivityLifecycleCallbacks;

import android.os.Bundle;
import android.os.RemoteException;

import java.util.ArrayList;
import java.util.List;

import android.util.Log;


public class ConsumerIrExtraActivityManager extends ReceiveCallback.Stub implements
        ActivityLifecycleCallbacks {

    /**
     * Constructor.
     * @param context
     * @param service
     */
    public ConsumerIrExtraActivityManager(IConsumerIrExtraService service) {
        mService = service;
    }

    // Tag
    private static final String TAG = "ConsumerIrExtra";

    // Service
    private IConsumerIrExtraService mService;

    // Receiver Lock
    private Object mMutex = new Object();
    // Activity List
    private List<ActivityState> mActivities = new ArrayList<ActivityState>();
    // Callback
    private ConsumerIrExtraManager.ReceivedCallback mCallback;
    // Application
    private Application mApp;

    // Activity State
    private static final class ActivityState {
        Activity activity;
        boolean isResume;
    }

    /**
     * Stop Receive.
     */
    public void stopReceive() {

        synchronized (mMutex) {
            if (mCallback == null) {
                return;
            } else {
                mActivities.clear();
                if(mApp == null) {
                    mCallback = null;
                }
                else {
                    mApp.unregisterActivityLifecycleCallbacks(this);
                    mApp = null;
                }
            }
        }

        try {
            mService.stopReceive();
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    /**
     * Starting receive IR patterns.
     * @param callback
     * @param activity
     * @param activities
     * @return
     */
    public boolean startReceive(ConsumerIrExtraManager.ReceivedCallback callback, Activity activity,
            Activity... activities) {

        if (callback == null) {
            throw new IllegalArgumentException("Callback of the receiver MUST not be null");
        }

        if (activity == null) {
            throw new IllegalArgumentException("Activity MUST not be null");
        }

        if (activity.getWindow().isDestroyed()) {
            throw new IllegalArgumentException("Activity is destoryed");
        }

        Application app = activity.getApplication();
        if (activities == null || activities.length == 0) {
        } else {
            for (Activity act : activities) {
                if (act.getWindow().isDestroyed()) {
                    throw new
                       IllegalArgumentException("Activity from activites list is destoryed");
                }
                if (app == act.getApplication()) {
                    continue;
                } else {
                    throw new
                       IllegalArgumentException("Application of activities must be the same");
                }
            }
        }

        synchronized (mMutex) {
            if (mCallback == null) {
                mCallback = callback;
                mApp = app;
            } else {
                return false;
            }
        }

        boolean status = false;
        try {
            status = mService.startReceive(this);
        } catch (RemoteException e) {
            e.printStackTrace();
        }

        if (status) {
            synchronized (mMutex) {
                mActivities.clear();
            }

            registerActivity(activity);
            registerActivities(activities);
            app.registerActivityLifecycleCallbacks(this);
            return true;
        } else {
            synchronized (mMutex) {
                mActivities.clear();
                mCallback = null;
                mApp = null;
            }
            return false;
        }
    }

    /**
     * Update Activity..
     * @param activity
     * @param isResume
     */
    private void updateActivity(Activity activity, boolean isResume) {

        synchronized (this) {
            for (ActivityState state : mActivities) {
                if (state.activity == activity) {
                    state.isResume = isResume;
                    return;
                }
            }
        }
    }

    /**
     * Register Activity in List..
     * @param activity
     */
    private void registerActivity(Activity activity) {

        if (activity.getWindow().isDestroyed()) {
            Log.e(TAG, "[ConsumerIrActivityManger] registerActivity(), destoryed");
            return;
        }

        ActivityState state = new ActivityState();
        state.activity = activity;
        state.isResume = activity.isResumed();

        synchronized (this) {
            mActivities.add(state);
        }
    }

    /**
     * Register activity in list.
     * @param activity
     */
    private void registerActivities(Activity... activities) {

        if (activities == null) {
            return;
        }

        synchronized (this) {
            for (Activity activity : activities) {
                if (activity.getWindow().isDestroyed()) {
                    Log.e(TAG, "[ConsumerIrActivityManger] registerActivity(), destoryed");
                    continue;
                }

                ActivityState state = new ActivityState();
                state.activity = activity;
                state.isResume = activity.isResumed();
                mActivities.add(state);
            }
        }
    }

    /**
     * Remove activity in list.
     * @param activity
     */
    private void unregisterActivity(Activity activity) {

        synchronized (this) {
            for (ActivityState state : mActivities) {
                if (state.activity == activity) {
                    mActivities.remove(state);
                }
            }
        }
    }

    /**
     * Check if need to stop learning.
     * @return
     */
    private boolean needStop() {
        synchronized (this) {
            for (ActivityState state : mActivities) {
                if (state.isResume) {
                    return false;
                }
            }
        }
        return true;
    }

    @Override
    public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
        // Do nothing
    }

    @Override
    public void onActivityStarted(Activity activity) {
        // Do nothing
    }

    @Override
    public void onActivityResumed(Activity activity) {
        updateActivity(activity, true);
    }

    @Override
    public void onActivityPaused(Activity activity) {
        updateActivity(activity, false);
        if (needStop()) {
            stopReceive();
        }
    }

    @Override
    public void onActivityStopped(Activity activity) {
        // Do nothing
    }

    @Override
    public void onActivitySaveInstanceState(Activity activity, Bundle outState) {
        // Do nothing
    }

    @Override
    public void onActivityDestroyed(Activity activity) {
        unregisterActivity(activity);
        if (needStop()) {
            stopReceive();
        }
    }

    @Override
    public void onReceive(int carreerFrequency, int[] patterns, int size) throws RemoteException {
        ConsumerIrExtraManager.ReceivedCallback callback = null;
        synchronized (mMutex) {
            if (mCallback == null) {
                Log.d(TAG, "[ConsumerIrActivityManger] No callback defined");
                return;
            }
            callback = mCallback;
            mCallback = null;
        }

        callback.onReceive(carreerFrequency, patterns, size);
    }

    @Override
    public void onReceiveFailure(int status) throws RemoteException {
        ConsumerIrExtraManager.ReceivedCallback callback = null;
        synchronized (mMutex) {
            if (mCallback == null) {
                Log.d(TAG, "[ConsumerIrActivityManger] No callback defined");
                return;
            }
            callback = mCallback;
            mCallback = null;
        }

        callback.onReceiveFailure(status);
    }
}
