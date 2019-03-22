---
title: rom7.0-phone 电话相关
category:
  - Anrom
  - rom7.0
tags:
  - Android
  - Anrom
description: "电话项目相关记录"
date: 2017-08-30 16:40:17
---
## 自动录音状态 ##
1. 我方和a通话，b来电，我方接听，没有让a加入通话过程。
当我方和a通话时，我方收到一个b来电，在我方接听b来电之前，和a之间的录音都是保持的。
接听b来电后，录音结束。
挂断b来电，恢复和a通话时，重新建立一个新的录音文件，并录下此后的电话。

2. 我方和a通话，a接到b来电，a接听了，并将我方加入通话中
全程都有录音，包括“您好请不要挂机，对方正在通话中”这样的提示。也包括后来的三方通话录音。

```
frameworks/opt/telephone/src/java/com/android/internal/telephony/CallManager.java
frameworks/opt/telephone/src/java/com/android/internal/telephony/Phone.java

packages/services/Telephony/src/com/android/phone/PhoneGlobals.java
```

EVENT_CALL_WAITING
```
chris@chris-office ~/rom7.0/anrom7.0/frameworks $ grep -r "EVENT_CALL_WAITING"
opt/telephony/src/java/com/android/internal/telephony/CallTracker.java:    protected static final int EVENT_CALL_WAITING_INFO_CDMA        = 15;
opt/telephony/src/java/com/android/internal/telephony/GsmCdmaCallTracker.java:            mCi.registerForCallWaitingInfo(this, EVENT_CALL_WAITING_INFO_CDMA, null);
opt/telephony/src/java/com/android/internal/telephony/GsmCdmaCallTracker.java:            case EVENT_CALL_WAITING_INFO_CDMA:
opt/telephony/src/java/com/android/internal/telephony/GsmCdmaCallTracker.java:                        Rlog.d(LOG_TAG, "Event EVENT_CALL_WAITING_INFO_CDMA Received");
opt/telephony/src/java/com/android/internal/telephony/CallManager.java:    private static final int EVENT_CALL_WAITING = 108;
opt/telephony/src/java/com/android/internal/telephony/CallManager.java:        phone.registerForCallWaiting(handler, EVENT_CALL_WAITING, null);
opt/telephony/src/java/com/android/internal/telephony/CallManager.java:                case EVENT_CALL_WAITING:
opt/telephony/src/java/com/android/internal/telephony/CallManager.java:                    if (VDBG) Rlog.d(LOG_TAG, " handleMessage (EVENT_CALL_WAITING)");
opt/telephony/tests/telephonytests/src/com/android/internal/telephony/CallManagerTest.java:        Field field = CallManager.class.getDeclaredField("EVENT_CALL_WAITING");

```

```
chris@chris-office ~/rom7.0/anrom7.0/packages $ grep -r "EVENT_CALL_"
services/Telecomm/src/com/android/server/telecom/ConnectionServiceWrapper.java:                        call.onConnectionEvent(Connection.EVENT_CALL_MERGE_FAILED, null);
services/Telecomm/tests/src/com/android/server/telecom/tests/BasicCallTests.java:                eq(testCall2.mCallId), eq(Connection.EVENT_CALL_MERGE_FAILED), any(Bundle.class));
services/Telecomm/tests/src/com/android/server/telecom/tests/BasicCallTests.java:                eq(testCall2.mCallId), eq(Connection.EVENT_CALL_MERGE_FAILED), any(Bundle.class));
services/Telephony/src/com/android/services/telephony/TelephonyConnection.java:            sendConnectionEvent(Connection.EVENT_CALL_PULL_FAILED, null);

```


```
http://blog.csdn.net/l173864930/article/details/12235513
```

## 状态码 ##
### frameworks/base/telecomm/java/android/telecom/Call.java ###
```
    /**
     * The state of a {@code Call} when newly created.
     */
    public static final int STATE_NEW = 0;

    /**
     * The state of an outgoing {@code Call} when dialing the remote number, but not yet connected.
     */
    public static final int STATE_DIALING = 1;

    /**
     * The state of an incoming {@code Call} when ringing locally, but not yet connected.
     */
    public static final int STATE_RINGING = 2;

    /**
     * The state of a {@code Call} when in a holding state.
     */
    public static final int STATE_HOLDING = 3;

    /**
     * The state of a {@code Call} when actively supporting conversation.
     */
    public static final int STATE_ACTIVE = 4;

    /**
     * The state of a {@code Call} when no further voice or other communication is being
     * transmitted, the remote side has been or will inevitably be informed that the {@code Call}
     * is no longer active, and the local data transport has or inevitably will release resources
     * associated with this {@code Call}.
     */
    public static final int STATE_DISCONNECTED = 7;

    /**
     * The state of an outgoing {@code Call} when waiting on user to select a
     * {@link PhoneAccount} through which to place the call.
     */
    public static final int STATE_SELECT_PHONE_ACCOUNT = 8;

    /**
     * @hide
     * @deprecated use STATE_SELECT_PHONE_ACCOUNT.
     */
    @Deprecated
    @SystemApi
    public static final int STATE_PRE_DIAL_WAIT = STATE_SELECT_PHONE_ACCOUNT;

    /**
     * The initial state of an outgoing {@code Call}.
     * Common transitions are to {@link #STATE_DIALING} state for a successful call or
     * {@link #STATE_DISCONNECTED} if it failed.
     */
    public static final int STATE_CONNECTING = 9;

    /**
     * The state of a {@code Call} when the user has initiated a disconnection of the call, but the
     * call has not yet been disconnected by the underlying {@code ConnectionService}.  The next
     * state of the call is (potentially) {@link #STATE_DISCONNECTED}.
     */
    public static final int STATE_DISCONNECTING = 10;

    /**
     * The state of an external call which is in the process of being pulled from a remote device to
     * the local device.
     * <p>
     * A call can only be in this state if the {@link Details#PROPERTY_IS_EXTERNAL_CALL} property
     * and {@link Details#CAPABILITY_CAN_PULL_CALL} capability are set on the call.
     * <p>
     * An {@link InCallService} will only see this state if it has the
     * {@link TelecomManager#METADATA_INCLUDE_EXTERNAL_CALLS} metadata set to {@code true} in its
     * manifest.
     */
    public static final int STATE_PULLING_CALL = 11;
```

### packages/services/Telecomm/src/com/android/server/telecom/CallState.java ###
```
    /**
     * Indicates that a call is new and not connected. This is used as the default state internally
     * within Telecom and should not be used between Telecom and call services. Call services are
     * not expected to ever interact with NEW calls, but {@link android.telecom.InCallService}s will
     * see calls in this state.
     */
    public static final int NEW = 0;

    /**
     * The initial state of an outgoing {@code Call}.
     * Common transitions are to {@link #DIALING} state for a successful call or
     * {@link #DISCONNECTED} if it failed.
     */
    public static final int CONNECTING = 1;

    /**
     * The state of an outgoing {@code Call} when waiting on user to select a
     * {@link android.telecom.PhoneAccount} through which to place the call.
     */
    public static final int SELECT_PHONE_ACCOUNT = 2;

    /**
     * Indicates that a call is outgoing and in the dialing state. A call transitions to this state
     * once an outgoing call has begun (e.g., user presses the dial button in Dialer). Calls in this
     * state usually transition to {@link #ACTIVE} if the call was answered or {@link #DISCONNECTED}
     * if the call was disconnected somehow (e.g., failure or cancellation of the call by the user).
     */
    public static final int DIALING = 3;

    /**
     * Indicates that a call is incoming and the user still has the option of answering, rejecting,
     * or doing nothing with the call. This state is usually associated with some type of audible
     * ringtone. Normal transitions are to {@link #ACTIVE} if answered or {@link #DISCONNECTED}
     * otherwise.
     */
    public static final int RINGING = 4;

    /**
     * Indicates that a call is currently connected to another party and a communication channel is
     * open between them. The normal transition to this state is by the user answering a
     * {@link #DIALING} call or a {@link #RINGING} call being answered by the other party.
     */
    public static final int ACTIVE = 5;

    /**
     * Indicates that the call is currently on hold. In this state, the call is not terminated
     * but no communication is allowed until the call is no longer on hold. The typical transition
     * to this state is by the user putting an {@link #ACTIVE} call on hold by explicitly performing
     * an action, such as clicking the hold button.
     */
    public static final int ON_HOLD = 6;
    /**
     * Indicates that a call is currently disconnected. All states can transition to this state
     * by the call service giving notice that the connection has been severed. When the user
     * explicitly ends a call, it will not transition to this state until the call service confirms
     * the disconnection or communication was lost to the call service currently responsible for
     * this call (e.g., call service crashes).
     */
    public static final int DISCONNECTED = 7;

    /**
     * Indicates that the call was attempted (mostly in the context of outgoing, at least at the
     * time of writing) but cancelled before it was successfully connected.
     */
    public static final int ABORTED = 8;

    /**
     * Indicates that the call is in the process of being disconnected and will transition next
     * to a {@link #DISCONNECTED} state.
     * <p>
     * This state is not expected to be communicated from the Telephony layer, but will be reported
     * to the InCall UI for calls where disconnection has been initiated by the user but the
     * ConnectionService has confirmed the call as disconnected.
     */
    public static final int DISCONNECTING = 9;

    /**
     * Indicates that the call is in the process of being pulled to the local device.
     * <p>
     * This state should only be set on a call with
     * {@link android.telecom.Connection#PROPERTY_IS_EXTERNAL_CALL} and
     * {@link android.telecom.Connection#CAPABILITY_CAN_PULL_CALL}.
     */
    public static final int PULLING = 10;
```

### packages/apps/Dialer/InCallUI/src/com/android/incallui/Call.java ###
```
public class Call {
    /* Defines different states of this call */
    public static class State {
        public static final int INVALID = 0;
        public static final int NEW = 1;            /* The call is new. */
        public static final int IDLE = 2;           /* The call is idle.  Nothing active */
        public static final int ACTIVE = 3;         /* There is an active call */
        public static final int INCOMING = 4;       /* A normal incoming phone call */
        public static final int CALL_WAITING = 5;   /* Incoming call while another is active */
        public static final int DIALING = 6;        /* An outgoing call during dial phase */
        public static final int REDIALING = 7;      /* Subsequent dialing attempt after a failure */
        public static final int ONHOLD = 8;         /* An active phone call placed on hold */
        public static final int DISCONNECTING = 9;  /* A call is being ended. */
        public static final int DISCONNECTED = 10;  /* State after a call disconnects */
        public static final int CONFERENCED = 11;   /* Call part of a conference call */
        public static final int SELECT_PHONE_ACCOUNT = 12; /* Waiting for account selection */
        public static final int CONNECTING = 13;    /* Waiting for Telecom broadcast to finish */
        public static final int BLOCKED = 14;       /* The number was found on the block list */
```

### packages/apps/Dialer/InCallUI/src/com/android/incallui/InCallPresenter.java ###
```
    public enum InCallState {
        // InCall Screen is off and there are no calls
        NO_CALLS,

        // Incoming-call screen is up
        INCOMING,

        // In-call experience is showing
        INCALL,

        // Waiting for user input before placing outgoing call
        WAITING_FOR_ACCOUNT,

        // UI is starting up but no call has been initiated yet.
        // The UI is waiting for Telecom to respond.
        PENDING_OUTGOING,

        // User is dialing out
        OUTGOING;

        public boolean isIncoming() {
            return (this == INCOMING);
        }

        public boolean isConnectingOrConnected() {
            return (this == INCOMING ||
                    this == OUTGOING ||
                    this == INCALL);
        }
    }
```

## 调用路线 ##
### state ###
```
base/telecomm/java/android/telecom/Call.java:1368     private static String stateToString(int state) {    <-
base/telecomm/java/android/telecom/Call.java:1353     public String toString() {    <-
```


### mState ###
** 写入： **
#### 1. base/telecomm/java/android/telecom/Call.java:1433 ####
```
    Call(Phone phone, String telecomCallId, InCallAdapter inCallAdapter, int state) {
        mPhone = phone;
        mTelecomCallId = telecomCallId;
        mInCallAdapter = inCallAdapter;
        mState = state;
    }
```


#### 2. base/telecomm/java/android/telecom/Call.java:1471 ####
```
final void internalUpdate(ParcelableCall parcelableCall, Map<String, Call> callIdMap) {
.....
        if (stateChanged) {
            mState = state;
        }
......
}
```

* 2.1 framework/base/telecomm/java/android/telecom/Phone.java
```
final void internalAddCall(ParcelableCall parcelableCall)
和
final void internalUpdateCall(ParcelableCall parcelableCall) {
```

* 2.1.1 base/telecomm/java/android/telecom/InCallService.java:
```
 mPhone.internalAddCall((ParcelableCall) msg.obj);
和
 mPhone.internalUpdateCall((ParcelableCall) msg.obj);
```

## 状态码转换 ##
packages/services/Telecomm/src/com/android/server/telecom/CallState.java


### packages/apps/Dialer/InCallUI/src/com/android/incallui/Call.java ###
从 frameworks/base/telecomm/java/android/telecom/Call.java
向 packages/apps/Dialer/InCallUI/src/com/android/incallui/Call.java
```
    private static int translateState(int state) {
        switch (state) {
            case android.telecom.Call.STATE_NEW:
            case android.telecom.Call.STATE_CONNECTING:
                return Call.State.CONNECTING;
            case android.telecom.Call.STATE_SELECT_PHONE_ACCOUNT:
                return Call.State.SELECT_PHONE_ACCOUNT;
            case android.telecom.Call.STATE_DIALING:
                return Call.State.DIALING;
            case android.telecom.Call.STATE_RINGING:
                return Call.State.INCOMING;
            case android.telecom.Call.STATE_ACTIVE:
                return Call.State.ACTIVE;
            case android.telecom.Call.STATE_HOLDING:
                return Call.State.ONHOLD;
            case android.telecom.Call.STATE_DISCONNECTED:
                return Call.State.DISCONNECTED;
            case android.telecom.Call.STATE_DISCONNECTING:
                return Call.State.DISCONNECTING;
            default:
                return Call.State.INVALID;
        }
    }
```

frameworks/base/telecomm/java/android/telecom/Call.java
```
private void fireStateChanged(final int newState) {
        for (CallbackRecord<Callback> record : mCallbackRecords) {
            final Call call = this;
            final Callback callback = record.getCallback();
            record.getHandler().post(new Runnable() {
                @Override
                public void run() {
                    callback.onStateChanged(call, newState);
                }
            });
        }
    }
```
创造一个CallbackRecord,并且注册到这里？？？？
注册函数：
```
public void registerCallback(Callback callback) {
        registerCallback(callback, new Handler());
    }

    /**
     * Registers a callback to this {@code Call}.
     *
     * @param callback A {@code Callback}.
     * @param handler A handler which command and status changes will be delivered to.
     */
    public void registerCallback(Callback callback, Handler handler) {
        unregisterCallback(callback);
        // Don't allow new callback registration if the call is already being destroyed.
        if (callback != null && handler != null && mState != STATE_DISCONNECTED) {
            mCallbackRecords.add(new CallbackRecord<Callback>(callback, handler));
        }
    }
```
调用他的地方：
```
    public void addListener(Listener listener) {
        registerCallback(listener);
    }
```

## 电话状态调用流程 ##
### packages/services/Telecomm/src/con/android/server/telecom/InCallController.java ###
1. 在这里打印的电话状态，可以用
```
private void updateCall(Call call, boolean videoProviderChanged) {
        if (!mInCallServices.isEmpty()) {
            Log.i(this, "chris InCallController.java updateCall {Call: %s}",
                        (call != null ? call.toString() :"null"));
```




## 录音 ##
### 代码位置 ###
packages/apps/Dialer/InCallUI/src/com/android/incallui/CallButtonPresesnter.java
packages/apps/Dialer/InCallUI/src/com/android/incallui/CallRecorder.java
packages/apps/Dialer/src/com/android/services/callrecorder/CallRecorderService.java

```
http://www.jizhuomi.com/android/example/354.html
http://blog.sina.com.cn/s/blog_89429f6d0100yald.html
http://blog.csdn.net/wonengxing/article/details/42488433
http://blog.csdn.net/cjh_android/article/details/51341004
```



### 0916 ###
目前看来，打log出现需求的状态的地方是可以拿到所有需要的状态的。
log：
packages/services/Telecomm/src/con/android/server/telecom/InCallController.java ###
```
private void updateCall(Call call, boolean videoProviderChanged) {
        if (!mInCallServices.isEmpty()) {
            Log.i(this, "chris InCallController.java updateCall {Call: %s}",
                        (call != null ? call.toString() :"null"));
```
调用它的位置应该是
```
    public void onCallStateChanged(Call call, int oldState, int newState) {
        Log.d("chris","InCallController.java onCallStateChanged "
                "calling function updateCall");
        updateCall(call);
    }
```
这个调用在class InCallController当中，这个类 extends CallsManagerListenerBase.
再看CallsManagerListenerBase.java  这个类implements CallsManager.CallsManagerListener 
再看CallsManagerListener，在CallsManager.java文件中，是一个interface类。
```
    public interface CallsManagerListener {
        void onCallAdded(Call call);
        void onCallRemoved(Call call);
        void onCallStateChanged(Call call, int oldState, int newState);
        void onConnectionServiceChanged(
                Call call,
                ConnectionServiceWrapper oldService,
                ConnectionServiceWrapper newService);
        void onIncomingCallAnswered(Call call);
        void onIncomingCallRejected(Call call, boolean rejectWithMessage, String textMessage);
        void onCallAudioStateChanged(CallAudioState oldAudioState, CallAudioState newAudioState);
        void onRingbackRequested(Call call, boolean ringback);
        void onIsConferencedChanged(Call call);
        void onIsVoipAudioModeChanged(Call call);
        void onVideoStateChanged(Call call, int previousVideoState, int newVideoState);
        void onCanAddCallChanged(boolean canAddCall);
        void onSessionModifyRequestReceived(Call call, VideoProfile videoProfile);
        void onHoldToneRequested(Call call);
        void onExternalCallChanged(Call call, boolean isExternalCall);
    }
```
所以，下一步，看看Dialer中有没有其他CallsManagerListener，模仿写法，把romrecorder做成CallsManagerListener,就可以直接判断state，然后录音。


另一种方法，看能不能在service/telecomm中调用到romrecorder，所以，就是看app/Dialer和service/telecomm谁能调用谁。




### CallList的调用流程 ###
packages/apps/Dialer/InCallUI/src/com/android/incallui/CallList.java
#### 打入电话 ####
1. log打印
'public void onIncoming(Call call, List<String> textMessages)'
这个方法是来电时的log打印。
```
09-16 16:19:23.611  3914  3914 I InCall  : CallList - onIncoming - [Call_14, INCOMING, [Capabilities: CAPABILITY_SUPPORT_HOLD CAPABILITY_RESPOND_VIA_TEXT CAPABILITY_MUTE CAPABILITY_CANNOT_DOWNGRADE_VIDEO_TO_AUDIO], [Properties:], children:[], parent:null, conferenceable:[], videoState:Audio Only, mSessionModificationState:0, VideoSettings:(CameraDir:-1), mIsActivSub:false]
```

2. 调用
它由public void (final android.telecom.Call telecomCall)

#### 打出电话 ####
private void (Call call) 
这个方法是打出的时候的log打印
```
09-16 15:19:11.039  3914  3914 I InCall  : CallList - onUpdate - [Call_8, ACTIVE, [Capabilities: CAPABILITY_HOLD CAPABILITY_SUPPORT_HOLD CAPABILITY_MUTE CAPABILITY_CANNOT_DOWNGRADE_VIDEO_TO_AUDIO], [Properties:], children:[], parent:null, conferenceable:[], videoState:Audio Only, mSessionModificationState:0, VideoSettings:(CameraDir:-1), mIsActivSub:false]
```
frameworks/base/telecomm/java/android/telecom/Call.java
```
private void fireStateChanged(final int newState) {
        for (CallbackRecord<Callback> record : mCallbackRecords) {
            final Call call = this;
            final Callback callback = record.getCallback();
            record.getHandler().post(new Runnable() {
                @Override
                public void run() {
                    callback.onStateChanged(call, newState);
                }
            });
        }
    }
```
创造一个CallbackRecord,并且注册到这里？？？？
注册函数：
```
public void registerCallback(Callback callback) {
        registerCallback(callback, new Handler());
    }

    /**
     * Registers a callback to this {@code Call}.
     *
     * @param callback A {@code Callback}.
     * @param handler A handler which command and status changes will be delivered to.
     */
    public void registerCallback(Callback callback, Handler handler) {
        unregisterCallback(callback);
        // Don't allow new callback registration if the call is already being destroyed.
        if (callback != null && handler != null && mState != STATE_DISCONNECTED) {
            mCallbackRecords.add(new CallbackRecord<Callback>(callback, handler));
        }
    }
```
调用他的地方：
```
    public void addListener(Listener listener) {
        registerCallback(listener);
    }
```

2. 调用
它由public void onUpdate(Call call)调用

### 要解决的问题 ###
**要解决申请权限的问题**
**要屏蔽掉原本电话中的录音功能**
**怎么通知app各种电话事件**



## 0917 ##
notify的处理方法
### 通知调用路径 ###
应用层
```
package com.pocketdigi.phonelistener;   
import android.app.Service;   
import android.content.BroadcastReceiver;   
import android.content.Context;   
import android.content.Intent;   
import android.telephony.PhoneStateListener;   
import android.telephony.TelephonyManager;   
public class PhoneReceiver extends BroadcastReceiver {   
@Override   
public void onReceive(Context context, Intent intent) {   
System.out.println("action"+intent.getAction());   
//如果是去电   
if(intent.getAction().equals(Intent.ACTION_NEW_OUTGOING_CALL)){   
String phoneNumber = intent   
.getStringExtra(Intent.EXTRA_PHONE_NUMBER);   
Log.d(TAG, "call OUT:" + phoneNumber);   
}else{   
//查了下android文档，貌似没有专门用于接收来电的action,所以，非去电即来电.   
//如果我们想要监听电话的拨打状况，需要这么几步 :   
* 第一：获取电话服务管理器TelephonyManager manager = this.getSystemService(TELEPHONY_SERVICE);   
* 第二：通过TelephonyManager注册我们要监听的电话状态改变事件。manager.listen(new MyPhoneStateListener(),   
* PhoneStateListener.LISTEN_CALL_STATE);这里的PhoneStateListener.LISTEN_CALL_STATE就是我们想要   
* 监听的状态改变事件，初次之外，还有很多其他事件哦。   
* 第三步：通过extends PhoneStateListener来定制自己的规则。将其对象传递给第二步作为参数。   
* 第四步：这一步很重要，那就是给应用添加权限。android.permission.READ_PHONE_STATE   
TelephonyManager tm = (TelephonyManager)context.getSystemService(Service.TELEPHONY_SERVICE);   
tm.listen(listener, PhoneStateListener.LISTEN_CALL_STATE);   
//设置一个监听器   
}   
}   
 listener=new PhoneStateListener(){   
@Override   
public void onCallStateChanged(int state, String incomingNumber) {   
//注意，方法必须写在super方法后面，否则incomingNumber无法获取到值。   
super.(state, incomingNumber);   
switch(state){   
case TelephonyManager.CALL_STATE_IDLE:   
System.out.println("挂断");   
break;   
case TelephonyManager.CALL_STATE_OFFHOOK:   
System.out.println("接听");   
break;   
case TelephonyManager.CALL_STATE_RINGING:   
System.out.println("响铃:来电号码"+incomingNumber);   
//输出来电号码   
break;   
}   
}   
};   
}   
```

尝试在PhoneStateListener中添加一个新方法，来获取真正的state。
/frameworks/base/telephony/java/android/telephony/PhoneStateListener.java




发出广播的地方
packages/services/Telecomm/src/com/android/server/telecom/PhoneStateBroadcaster.java
```
    @Override
    public void onCallStateChanged(Call call, int oldState, int newState) {
        if (call.isExternalCall()) {
            return;
        }
        updateStates(call);
    }
```
调用
```
    private void updateStates(Call call) {
        // Recalculate the current phone state based on the consolidated state of the remaining
        // calls in the call list.
        // Note: CallsManager#hasRingingCall() and CallsManager#getFirstCallWithState(..) do not
        // consider external calls, so an external call is going to cause the state to be idle.
        int callState = TelephonyManager.CALL_STATE_IDLE;
        if (mCallsManager.hasRingingCall()) {
            callState = TelephonyManager.CALL_STATE_RINGING;
        } else if (mCallsManager.getFirstCallWithState(CallState.DIALING, CallState.PULLING,
                CallState.ACTIVE, CallState.ON_HOLD) != null) {
            callState = TelephonyManager.CALL_STATE_OFFHOOK;
        }
        sendPhoneStateChangedBroadcast(call, callState);
    }
```
这里已经有Call,可以通过它取到CallState,文件位置：
packages/services/Telecomm/src/com/android/server/telecom/Call.java
packages/services/Telecomm/src/com/android/server/telecom/CallState.java








## notify收到的倒推调用 ##

* 
packages/services/Telecomm/src/com/android/server/telecom/TelecomServiceImpl.java
getCallState()
```
        /**
         * @see TelecomManager#getCallState
         */
        @Override
        public int getCallState() {
            try {
                Log.startSession("TSI.getCallState");
                synchronized (mLock) {
                    return mCallsManager.getCallState();
                }
            } finally {
                Log.endSession();
            }
        }
```

* packages/services/Telecomm/src/com/android/server/telecom/CallsManager.java
```
    /**
     * @return the call state currently tracked by {@link PhoneStateBroadcaster}
     */
    int getCallState() {
        return mPhoneStateBroadcaster.getCallState();
    }
```

* package/services/Telecomm/src/com/android/server/telecom/PhoneStateBroadcaster.java
```
    int getCallState() {
        return mCurrentState;
    }
```
既然return mCurrentState,那么就找找哪里set了mCurrentState,只有一处。
```
    private void sendPhoneStateChangedBroadcast(Call call, int phoneState) {
        if (phoneState == mCurrentState) {
            return;
        }

        mCurrentState = phoneState;

        String callHandle = null;
        if (call.getHandle() != null) {
            callHandle = call.getHandle().getSchemeSpecificPart();
        }

        try {
            if (mRegistry != null) {
                mRegistry.notifyCallState(phoneState, callHandle);
                Log.i(this, "Broadcasted state change: %s", mCurrentState);
            }
        } catch (RemoteException e) {
            Log.w(this, "RemoteException when notifying TelephonyRegistry of call state change.");
        }
    }
```
接下来，看看set过程的调用。就从这个sendPhoneStateChangedBroadcast开始倒推。
updateStates是唯一调用sendPhoneStateChangedBroadcast的地方，并且
**这就是把完整的state的转成只有三种state的地方**
```
    private void updateStates(Call call) {
        // Recalculate the current phone state based on the consolidated state of the remaining
        // calls in the call list.
        // Note: CallsManager#hasRingingCall() and CallsManager#getFirstCallWithState(..) do not
        // consider external calls, so an external call is going to cause the state to be idle.
        int callState = TelephonyManager.CALL_STATE_IDLE;
        if (mCallsManager.hasRingingCall()) {
            callState = TelephonyManager.CALL_STATE_RINGING;
        } else if (mCallsManager.getFirstCallWithState(CallState.DIALING, CallState.PULLING,
                CallState.ACTIVE, CallState.ON_HOLD) != null) {
            callState = TelephonyManager.CALL_STATE_OFFHOOK;
        }
        sendPhoneStateChangedBroadcast(call, callState);
    }
```
调用updateStates就比较多了，不好下手，还是考虑在sendPhoneStateChangedBroadcast中下手。
再回头看sendPhoneStateChangedBroadcast
```
    private void sendPhoneStateChangedBroadcast(Call call, int phoneState) {
        if (phoneState == mCurrentState) {
            return;
        }

        mCurrentState = phoneState;

        String callHandle = null;
        if (call.getHandle() != null) {
            callHandle = call.getHandle().getSchemeSpecificPart();
        }

        try {
            if (mRegistry != null) {
                mRegistry.notifyCallState(phoneState, callHandle);
                Log.i(this, "Broadcasted state change: %s", mCurrentState);
            }
        } catch (RemoteException e) {
            Log.w(this, "RemoteException when notifying TelephonyRegistry of call state change.");
        }
    }
```
其中有`mRegistry.notifyCallState(phoneState, callHandle);`
这个mRegistry的原型在frameworks/base/service/core/java/com/android/server/TelephonyRegistry.java
看看他的notifyCallState
```
    public void notifyCallState(int state, String incomingNumber) {
        if (!checkNotifyPermission("notifyCallState()")) {
            return;
        }

        if (VDBG) {
            log("notifyCallState: state=" + state + " incomingNumber=" + incomingNumber);
        }

        synchronized (mRecords) {
            for (Record r : mRecords) {
                if (r.matchPhoneStateListenerEvent(PhoneStateListener.LISTEN_CALL_STATE) &&
                        (r.subId == SubscriptionManager.DEFAULT_SUBSCRIPTION_ID)) {
                    try {
                        String incomingNumberOrEmpty = r.canReadPhoneState ? incomingNumber : "";
                        r.callback.onCallStateChanged(state, incomingNumberOrEmpty);
                    } catch (RemoteException ex) {
                        mRemoveList.add(r.binder);
                    }
                }
            }
            handleRemoveListLocked();
        }

        // Called only by Telecomm to communicate call state across different phone accounts. So
        // there is no need to add a valid subId or slotId.
        broadcastCallStateChanged(state, incomingNumber,
                SubscriptionManager.INVALID_PHONE_INDEX,
                SubscriptionManager.INVALID_SUBSCRIPTION_ID);
    }
```
其中有` r.callback.onCallStateChanged(state, incomingNumberOrEmpty);`
依样画葫芦，做一个notifyRealCallState，从sendPhoneStateChangedBroadcast中，把真实的state，给做一个notifyRealCallState,
**其中有一个checkNotifyPermission方法，考虑能不能用来做未来的应用验证**











## notify修改了 ##
frameworks/base/services/core/java/com/android/server/TelephonyRegistry.java

frameworks/base/telecomm/java/com/android/interrnal/telecom/ITelecomService.aidl

frameworks/base/telephony/java/android/telephony/PhoneStateListener.java
frameworks/base/telephony/java/android/telephony/TelephoneManager.java

frameworks/base/telephony/java/com/android/internal/telephony/ITelephonyRegistry.aidl
frameworks/base/telephony/java/com/android/internal/telephony/PhoneConstants.java

frameworks/opt/telephony/src/com/android/internal/telephony/Call.java
frameworks/opt/telephony/src/com/android/internal/telephony/DefaultPhoneNotifier.java
frameworks/opt/telephony/src/com/android/internal/telephony/Phone.java
frameworks/opt/telephony/src/com/android/internal/telephony/.java


package/service/Telecomm/src/com/android/server/telecom/TelecomServiceImpl.java
package/service/Telecomm/src/com/android/server/telecom/Call.java
package/service/Telecomm/src/com/android/server/telecom/CallsManager.java
package/service/Telecomm/src/com/android/server/telecom/CallsManagerListenerBase.java
package/service/Telecomm/src/com/android/server/telecom/CallState.java
package/service/Telecomm/src/com/android/server/telecom/PhoneStateBroadcaster.java


framewords/opt/telephony/src/java/com/android/internal/talephony/TelecomService.java

services/core/java/com/android/server/notification/NotificationManagerService.java

base/services/core/java/com/android/server/pm/DefaultPermissionGrantPolicy.java

data-binding/compiler/src/main/resources/api-versions.xml

frameworks/opt/telephony/src/java/com/android/internal/telephony/GsmCdmaPhone.java

## notify调用过程 ##
首先明确，
录音方法中的callstate，是packages/apps/Dialer/InCallUI/src/com/android/incallui/Call.java中的State。
而我们需要的notify的callstate只能自己造了。因为涉及到状态通知的两个地方能够拿到的是两种不同的state组合。
1. packages/services/Telecomm/src/com/android/server/telecom/CallState.java
```
    public static final int NEW = 0;
    public static final int CONNECTING = 1;
    public static final int SELECT_PHONE_ACCOUNT = 2;
    public static final int DIALING = 3;
    public static final int RINGING = 4;
    public static final int ACTIVE = 5;
    public static final int ON_HOLD = 6;
    public static final int DISCONNECTED = 7;
    public static final int ABORTED = 8;
    public static final int DISCONNECTING = 9;
    public static final int PULLING = 10;
```
2. framewords/opt/telephony/src/java/com/android/internal/telephony/Call.java
```
public enum State {
        IDLE, ACTIVE, HOLDING, DIALING, ALERTING, INCOMING, WAITING, DISCONNECTED, DISCONNECTING;

```
在packages/services/Telecomm/src/com/android/server/telecom/PhoneStateBroadcaster.java中，有一个把完整的CallState转成只有三种状态的方法。
在framewords/opt/telephony/src/java/com/android/internal/telephony/Call.java中，有把Call.State简单转成其他类型的启示代码。

**自己造的callstate**
在frameworks/base/telephony/java/android/telephony/TelephoyManager.java中创造以下callstate
```
    public static final int REAL_CALL_STATE_IDLE = 0;
    public static final int REAL_CALL_STATE_ACTIVE = 1;
    public static final int REAL_CALL_STATE_HOLDING = 3;
    public static final int REAL_CALL_STATE_RINGING = 4;
    public static final int REAL_CALL_STATE_DIALING = 5;
    //~ public static final int REAL_CALL_STATE_ALERTING = 5;
    //~ public static final int REAL_CALL_STATE_INCOMING = 6;
    //~ public static final int REAL_CALL_STATE_WAITING = 7;
    public static final int REAL_CALL_STATE_DISCONNECTED = 6;
    public static final int REAL_CALL_STATE_DISCONNECTING = 7;
    
```
1. 在frameworks/opt/telephony/src/java/com/android/internal/telephony/Call.java中写转换函数
```
    public int getRealState() {
        State state = getState();

        switch (state) {
            case IDLE:
                return TelephonyManager.REAL_CALL_STATE_IDLE;
            case ACTIVE:
                return TelephonyManager.REAL_CALL_STATE_ACTIVE;
            case HOLDING:
                return TelephonyManager.REAL_CALL_STATE_HOLDING;
            case INCOMING:
            case WAITING:
                return TelephonyManager.REAL_CALL_STATE_RINGING;
            case DIALING:
            case ALERTING:
                return TelephonyManager.REAL_CALL_STATE_DIALING;
            case DISCONNECTED:
                return TelephonyManager.REAL_CALL_STATE_DISCONNECTED;
            case DISCONNECTING:
                return TelephonyManager.REAL_CALL_STATE_DISCONNECTING;
            default:
                return TelephonyManager.REAL_CALL_STATE_IDLE;
        }
        return TelephonyManager.REAL_CALL_STATE_IDLE;
    }
```
2. 在packages/services/Telecomm/src/com/android/server/telecom/Call.java中写转换函数
```
public int getRealState() {
        int state = getState();

        switch (state) {
            case NEW:
            case CONNECTING:
            case SELECT_PHONE_ACCOUNT:
            case ABORTED:
            case PULLING:
                Log.i(LOG_TAG,"noti pacCall state:" + TelephonyManager.REAL_CALL_STATE_IDLE);
                return TelephonyManager.REAL_CALL_STATE_IDLE;
            case DIALING:
            Log.i(LOG_TAG,"noti pacCall state:" + TelephonyManager.REAL_CALL_STATE_DIALING);
                return TelephonyManager.REAL_CALL_STATE_DIALING
            case RINGING:
            Log.i(LOG_TAG,"noti pacCall state:" + TelephonyManager.REAL_CALL_STATE_RINGING);
                return TelephonyManager.REAL_CALL_STATE_RINGING;
            case ACTIVE:
            Log.i(LOG_TAG,"noti pacCall state:" + TelephonyManager.REAL_CALL_STATE_ACTIVE);
                return TelephonyManager.REAL_CALL_STATE_ACTIVE;
            case ON_HOLD:
            Log.i(LOG_TAG,"noti pacCall state:" + TelephonyManager.REAL_CALL_STATE_HOLDING);
                return TelephonyManager.REAL_CALL_STATE_HOLDING;
            case DISCONNECTED:
            Log.i(LOG_TAG,"noti pacCall state:" + TelephonyManager.REAL_CALL_STATE_DISCONNECTED);
                return TelephonyManager.REAL_CALL_STATE_DISCONNECTED;
            case DISCONNECTING:
            Log.i(LOG_TAG,"noti pacCall state:" + TelephonyManager.REAL_CALL_STATE_DISCONNECTING);
                return TelephonyManager.REAL_CALL_STATE_DISCONNECTING;
            default:
            Log.i(LOG_TAG,"noti pacCall state:default");
                return TelephonyManager.REAL_CALL_STATE_IDLE;
        }
        Log.i(LOG_TAG,"noti pacCall state:nomatch");
        return TelephonyManager.REAL_CALL_STATE_IDLE;
    }
```

### 调用过程/修改路径 ##
1. frameworks/base/telephony/java/android/telephony/PhoneStateListener.java
```
        public void onRealCallStateChanged(int state, String incomingNumber) {
            log("PhoneStateListener onRealCallStateChanged");
            send(LISTEN_REAL_CALL_STATE, state, 0, incomingNumber);
        }
```
来自于
```
    public PhoneStateListener(int subId, Looper looper) {
        if (DBG) log("ctor: subId=" + subId + " looper=" + looper);
        mSubId = subId;
        mHandler = new Handler(looper) {
            public void handleMessage(Message msg) {
                if (DBG) {
                    log("mSubId=" + mSubId + " what=0x" + Integer.toHexString(msg.what)
                            + " msg=" + msg);
                }
                switch (msg.what) {
.......................
                    case LISTEN_CALL_STATE:
                        log("PhoneStateListener case LISTEN_CALL_STATE");
                        PhoneStateListener.this.onCallStateChanged(msg.arg1, (String)msg.obj);
                        break;
                    //add by chris
                    case LISTEN_REAL_CALL_STATE:
                        log("PhoneStateListener case LISTEN_REAL_CALL_STATE");
                        PhoneStateListener.this.onRealCallStateChanged(msg.arg1, (String)msg.obj);
                        break;

```
查找一下LISTEN_CALL_STATE，后续到
2. frameworks/base/services/core/java/com/android/server/TelephonyRegistry.java
LISTEN_CALL_STATE主要用在了权限判断和条件筛选
抄了notifyCallState 和 notifyCallStateForPhoneId 为 notifyRealCallState 和 notifyRealCallStateForPhoneId
同时，在frameworks/base/telephony/java/com/android/internal/telephony/ITelephonyRegistry.aidl中也要作相应修改
这里面的incomingNumber直接给了真实的incomingNumber，要注意，这里传入的state会被送去给r.callback.onRealCallStateChanged(state, incomingNumberOrEmpty);所以，要确保，调用这个函数的时候，给的state就是真实的state

3. 先看notifyCallState的调用
	1. packages/services/Telecomm/src/com/android/server/telecom/PhoneStateBroadcaster.java:114: mRegistry.notifyCallState(phoneState, callHandle);
```
    private void sendPhoneStateChangedBroadcast(Call call, int phoneState) {
        if (phoneState == mCurrentState) {
            return;
        }

        mCurrentState = phoneState;

        String callHandle = null;
        if (call.getHandle() != null) {
            callHandle = call.getHandle().getSchemeSpecificPart();
        }

        try {
            if (mRegistry != null) {
        //在这里
                mRegistry.(phoneState, callHandle);
        //增加notifyRealCallState
        		mRegistry.notifyRealCallState(call.getState(), callHandle);
                
                Log.i(this, "Broadcasted state change: %s", mCurrentState);
            }
        } catch (RemoteException e) {
            Log.w(this, "RemoteException when notifying TelephonyRegistry of call state change.");
        }
    }
```
在这里已经有Call了，可以直接使用Call.getRealState()来获取转换好的状态
所以，直接增加mRegistry.notifyRealCallState(call.getRealState(), callHandle);

4. 再看notifyCallStateForPhoneId的调用
	1. framewords/opt/telephony/src/java/com/android/internal/telephony/DefaultPhoneNotifier.java:66: mRegistry.notifyCallStateForPhoneId(phoneId, subId,
```
    @Override
    public void notifyPhoneState(Phone sender) {
        Rlog.d(LOG_TAG, "DefaultPhoneNotifier notifyPhoneState")
        Call ringingCall = sender.getRingingCall();
        int subId = sender.getSubId();
        int phoneId = sender.getPhoneId();
        String incomingNumber = "";
        if (ringingCall != null && ringingCall.getEarliestConnection() != null) {
            incomingNumber = ringingCall.getEarliestConnection().getAddress();
        }
        try {
            if (mRegistry != null) {
                Rlog.d(LOG_TAG, "noti DefaultPhoneNotifier notifyPhoneState: mRegistry="
                    + mRegistry + " ss=" + sender.getSignalStrength() + " sender=" + sender
                    + " phoneId" + phoneId + " subId" + subId + " sender.getsate()"
                    + sender.getState() + " incomingNumber" + incomingNumber);
                //在这里
                mRegistry.notifyCallStateForPhoneId(phoneId, subId,
                        convertCallState(sender.getState()), incomingNumber);
                Rlog.d(LOG_TAG, "noti DefaultPhoneNotifier notifyRealCallStateForPhoneId: mRegistry="
                    + mRegistry + " ss=" + sender.getSignalStrength() + " sender=" + sender
                    + " phoneId" + phoneId + " subId" + subId + " ringingCall.getState()"
                    + ringingCall.getState() + " incomingNumber" + incomingNumber);
                //加上这个
                mRegistry.notifyRealCallStateForPhoneId(phoneId, subId,
                        ringingCall.getRealState(), incomingNumber);
            }
        } catch (RemoteException ex) {
            // system process is dead
        }
    }
```
同样增加notifyRealCallStateForPhoneId,使用getRealState()

**修改permission**
frameworks/base/core/res/AndroidManifest.xml
