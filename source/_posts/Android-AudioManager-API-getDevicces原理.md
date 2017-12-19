---
title: 'Android AudioManager API: getDevicces原理'
date: 2017-12-16 15:24:58
tags: Android
categories: Android Audio
---
阅读Android开发者文档，我们看到在AudioManager中判断有线耳机和A2DP的是否存在有两组API，分别是isWiredHeadsetOn()，isBluetoothA2dpOn()和getDevices(int)，而isWiredHeadsetOn()在API 14中废弃，isBluetoothA2dpOn()在API 26中废弃，google建议都使用getDevices(int)，而在使用getDevices(int)过程中遇到bug，所以本文就研究一下getDevices(int)的原理。

## 宏观剖析getDevices
首先我们通过一张图，从进程的角度看看getDevices API的原理。
![](https://i.imgur.com/DfSkS3c.png)
我们可以将getDevices大概分为两种情况讨论，第一次调用：
1. Client调用AudioManager的getDevices函数，获取MainLooper，创建Handler。
2. 获取AudioPolicyService的代理，创建AudioPolicyServiceClient，添加回调AudioPortCallback。
3. 将AudioPolicyServiceClien注册到AudioPolicyService，更新cache，获取可用设备list返回。

我们再看第二种情况，在有新设备和手机连接的时候：
4. 此时会进入AudioPolicyService进程中，调用AudioPolicyManager的setDeviceConnectionStateInt函数。
5. 然后调用AudioPolicyClient的onAudioPortListUpdate，通过AudioPolicyService启动一文，我们知道，最终会调用AudioPolicyService的onAudioPortListUpdate
6. 调用所有AudioPolicyServiceClien的AudioPortCallback，获取之前创建的Handler。
7. 发送设备更新消息。
8. 添加消息到消息队列。
9. Looper取出消息队列中的消息。
10. 处理设备更新消息。
11. 后续该Client调用getDevices重新获取设备list，更新cache。

## 时序图
首先我们看看第一次调用getDevices的过程，在这个过程中，主要由两个工作，一是注册回调通知，二是获取设备列表；后面再次调用就不需要再次注册了，直接获取设备列表即可。
![](https://i.imgur.com/wCmfSpu.png)
下面是当设备变化时的回调通知过程。
![](https://i.imgur.com/K9B1xp2.png)
## 微观剖析getDevices
### Java层调用
在AudioManager中会调用从getDevicesStatic，listAudioDevicePorts到updateAudioPortCache，由于主要的处理都在updateAudioPortCache中，所以我们主要看一下该函数，在其中先初始化AudioPortEventHandler，若sAudioPortGeneration是初始值，则更新sAudioPortsCached，并返回其中的值，否则直接返回sAudioPortsCached中的值。
```java
    public AudioDeviceInfo[] getDevices(int flags) {
        return getDevicesStatic(flags);
    }

    public static AudioDeviceInfo[] getDevicesStatic(int flags) {
        ......
        int status = AudioManager.listAudioDevicePorts(ports);
        ......
    }

    public static int listAudioDevicePorts(ArrayList<AudioDevicePort> devices) {
        ......
        int status = updateAudioPortCache(ports, null, null);
        ......
    }

    static int resetAudioPortGeneration() {
        int generation;
        synchronized (sAudioPortGeneration) {
            generation = sAudioPortGeneration;
            sAudioPortGeneration = AUDIOPORT_GENERATION_INIT;
        }
        return generation;
    }

    static int updateAudioPortCache(ArrayList<AudioPort> ports, ArrayList<AudioPatch> patches,
                                    ArrayList<AudioPort> previousPorts) {
        sAudioPortEventHandler.init();
        synchronized (sAudioPortGeneration) {
            if (sAudioPortGeneration == AUDIOPORT_GENERATION_INIT) {
                ......

                do {
                    newPorts.clear();
                    status = AudioSystem.listAudioPorts(newPorts, portGeneration);
                    ......
                } while (patchGeneration[0] != portGeneration[0]);
                ......
                sAudioPortGeneration = portGeneration[0];
            }
            if (ports != null) {
                ports.clear();
                ports.addAll(sAudioPortsCached);
            }
            ......
        }
        return SUCCESS;
    }
```
在updateAudioPortCache会初始化AudioPortEventHandler，下面我们看看具体会做哪些事情。首先会获取当前进程的MainLooper，并以此Looper创建Handler，重写Handler消息处理函数。然后调用native_setup进入JNI层。而handleMessage对设备更新消息的处理是resetAudioPortGeneration，这样就会是的下次调用getDevices会更新sAudioPortsCached，从而得到更新后的设备列表。
```java
class AudioPortEventHandler {
    void init() {
        synchronized (this) {
            if (mHandler != null) {
                return;
            }
            // find the looper for our new event handler
            Looper looper = Looper.getMainLooper();
            if (looper != null) {
                mHandler = new Handler(looper) {
                    @Override
                    public void handleMessage(Message msg) {
                        ......
                        if (msg.what == AUDIOPORT_EVENT_PORT_LIST_UPDATED ||
                                msg.what == AUDIOPORT_EVENT_PATCH_LIST_UPDATED ||
                                msg.what == AUDIOPORT_EVENT_SERVICE_DIED) {
                            AudioManager.resetAudioPortGeneration();
                        }
                        if (listeners.isEmpty()) {
                            return;
                        }
                        ArrayList<AudioPort> ports = new ArrayList<AudioPort>();
                        ArrayList<AudioPatch> patches = new ArrayList<AudioPatch>();
                        if (msg.what != AUDIOPORT_EVENT_SERVICE_DIED) {
                            int status = AudioManager.updateAudioPortCache(ports, patches, null);
                            if (status != AudioManager.SUCCESS) {
                                return;
                            }
                        }
                        ......
                    }
                };
                native_setup(new WeakReference<AudioPortEventHandler>(this));
            } else {
                mHandler = null;
            }
        }
    }

    private native void native_setup(Object module_this);

    @SuppressWarnings("unused")
    private static void postEventFromNative(Object module_ref,
                                            int what, int arg1, int arg2, Object obj) {
        AudioPortEventHandler eventHandler =
                (AudioPortEventHandler)((WeakReference)module_ref).get();
        if (eventHandler == null) {
            return;
        }
        if (eventHandler != null) {
            Handler handler = eventHandler.handler();
            if (handler != null) {
                Message m = handler.obtainMessage(what, arg1, arg2, obj);
                handler.sendMessage(m);
            }
        }
    }
}
```
### JNI层调用
仔细追代码，发现AudioPortEventHandler的JNI函数定义在android_media_AudioSystem.cpp。且Java实例方法native_setup对应着JNI层android_media_AudioSystem_eventHandlerSetup。同时还会有Java静态方法postEventFromNative和JNI层的gAudioPortEventHandlerMethods.postEventFromNative的对应。
```cpp
static const char* const kEventHandlerClassPathName =
        "android/media/AudioPortEventHandler";

static const JNINativeMethod gEventHandlerMethods[] = {
    {"native_setup",
        "(Ljava/lang/Object;)V",
        (void *)android_media_AudioSystem_eventHandlerSetup},
    {"native_finalize",
        "()V",
        (void *)android_media_AudioSystem_eventHandlerFinalize},
};

int register_android_media_AudioSystem(JNIEnv *env) {
   ......
   jclass eventHandlerClass = FindClassOrDie(env, kEventHandlerClassPathName);
   gAudioPortEventHandlerMethods.postEventFromNative = GetStaticMethodIDOrDie(
                                       env, eventHandlerClass, 
                                       "postEventFromNative",
                                       "(Ljava/lang/Object;IIILjava/lang/Object;)V");
   ......
   return RegisterMethodsOrDie(env, kEventHandlerClassPathName, gEventHandlerMethods,
                     NELEM(gEventHandlerMethods));
}
```
在android_media_AudioSystem_eventHandlerSetup中会创建JNIAudioPortCallback，然后调用AudioSystem的addAudioPortCallback函数进行注册添加。后续有设备的变化，会通过这个回调sendEvent，进而通过gAudioPortEventHandlerMethods.postEventFromNative返回Java层，通过Handler处理消息。
```cpp
static void
android_media_AudioSystem_eventHandlerSetup(JNIEnv *env, jobject thiz, jobject weak_this)
{
    ALOGV("eventHandlerSetup");
    //JNIAudioPortCallback从AudioPortCallback继承
    sp<JNIAudioPortCallback> callback = new JNIAudioPortCallback(env, thiz, weak_this);
    if (AudioSystem::addAudioPortCallback(callback) == NO_ERROR) {
        setJniCallback(env, thiz, callback);
    }
}

void JNIAudioPortCallback::sendEvent(int event)
{
    JNIEnv *env = AndroidRuntime::getJNIEnv();
    if (env == NULL) {
        return;
    }
    env->CallStaticVoidMethod(mClass, gAudioPortEventHandlerMethods.postEventFromNative, mObject,
                              event, 0, 0, NULL);
    if (env->ExceptionCheck()) {
        ALOGW("An exception occurred while notifying an event.");
        env->ExceptionClear();
    }
}
void JNIAudioPortCallback::onAudioPortListUpdate()
{
    sendEvent(AUDIOPORT_EVENT_PORT_LIST_UPDATED);
}
```
### Native层调用
AudioSystem的addAudioPortCallback会先获取AudioPolicyService的代理，同时创建AudioPolicyServiceClient，然后将AudioPolicyServiceClient注册到AudioPolicyService，在AudioPolicyService中，以NotificationClien封装AudioPolicyServiceClient，再将其以UID为key保存在mNotificationClients中。然后向AudioPolicyServiceClient中添加回调AudioPortCallback。
```cpp
const sp<IAudioPolicyService> AudioSystem::get_audio_policy_service()
{
    sp<IAudioPolicyService> ap;
    sp<AudioPolicyServiceClient> apc;
    {
        Mutex::Autolock _l(gLockAPS);
        if (gAudioPolicyService == 0) {
            sp<IServiceManager> sm = defaultServiceManager();
            sp<IBinder> binder;
            do {
                binder = sm->getService(String16("media.audio_policy"));
                if (binder != 0)
                    break;
                ALOGW("AudioPolicyService not published, waiting...");
                usleep(500000); // 0.5 s
            } while (true);
            if (gAudioPolicyServiceClient == NULL) {
                gAudioPolicyServiceClient = new AudioPolicyServiceClient();
            }
            binder->linkToDeath(gAudioPolicyServiceClient);
            gAudioPolicyService = interface_cast<IAudioPolicyService>(binder);
            LOG_ALWAYS_FATAL_IF(gAudioPolicyService == 0);
            apc = gAudioPolicyServiceClient;
            // Make sure callbacks can be received by gAudioPolicyServiceClient
            ProcessState::self()->startThreadPool();
        }
        ap = gAudioPolicyService;
    }
    if (apc != 0) {
        ap->registerClient(apc);
    }
    return ap;
}

status_t AudioSystem::addAudioPortCallback(const sp<AudioPortCallback>& callback)
{
    const sp<IAudioPolicyService>& aps = AudioSystem::get_audio_policy_service();
    if (aps == 0) return PERMISSION_DENIED;
    Mutex::Autolock _l(gLockAPS);
    if (gAudioPolicyServiceClient == 0) {
        return NO_INIT;
    }
    int ret = gAudioPolicyServiceClient->addAudioPortCallback(callback);
    if (ret == 1) {
        aps->setAudioPortCallbacksEnabled(true);
    }
    return (ret < 0) ? INVALID_OPERATION : NO_ERROR;
}

int AudioSystem::AudioPolicyServiceClient::addAudioPortCallback(
        const sp<AudioPortCallback>& callback)
{
    Mutex::Autolock _l(mLock);
    for (size_t i = 0; i < mAudioPortCallbacks.size(); i++) {
        if (mAudioPortCallbacks[i] == callback) {
            return -1;
        }
    }
    mAudioPortCallbacks.add(callback);
    return mAudioPortCallbacks.size();
}
```
如下就是AudioPolicyService保存AudioPolicyServiceClient的代码，我们看到会判断key值uid是否存在，若已经存在，则不会重复保存。**我们知道在Android中PID是唯一的，而UID是不唯一的，因为进程之前可以通过sharedUid共享数据。所以从这点看，这里是存在隐患的，假如有两个进程共享UID，且两个进程都会使用getDevices注册回调，这就有可能是的后注册的回调无法得到调用而得不到设备列表更新。**
```cpp
void AudioPolicyService::registerClient(const sp<IAudioPolicyServiceClient>& client)
{
    if (client == 0) {
        ALOGW("%s got NULL client", __FUNCTION__);
        return;
    }
    Mutex::Autolock _l(mNotificationClientsLock);
    uid_t uid = IPCThreadState::self()->getCallingUid();
    if (mNotificationClients.indexOfKey(uid) < 0) {
        sp<NotificationClient> notificationClient = new NotificationClient(this,
                                                                           client,
                                                                           uid);
        ALOGV("registerClient() client %p, uid %d", client.get(), uid);
        mNotificationClients.add(uid, notificationClient);
        sp<IBinder> binder = IInterface::asBinder(client);
        binder->linkToDeath(notificationClient);
    }
}

void AudioPolicyService::setAudioPortCallbacksEnabled(bool enabled)
{
    Mutex::Autolock _l(mNotificationClientsLock);
    uid_t uid = IPCThreadState::self()->getCallingUid();
    if (mNotificationClients.indexOfKey(uid) < 0) {
        return;
    }
    mNotificationClients.valueFor(uid)->setAudioPortCallbacksEnabled(enabled);
}
```
如下是当有设备更新的时候，反向调用回调的过程，设备接入或移除，都会经过AudioPolicyManager的setDeviceConnectionStateInt，然后调用AudioPolicyClient的onAudioPortListUpdate，在AudioPolicyService启动过程我们知道，AudioPolicyClient中的调用几乎最终都会进入AudioPolicyService，使用AudioCommandThread异步处理，调用NotificationClients中的onAudioPortListUpdate，再到AudioPolicyServiceClient的onAudioPortListUpdate，再到JNIAudioPortCallback的onAudioPortListUpdate，调用Java静态函数postEventFromNative，通过Handler发消息执行reset，从而通知所有的client连接的设备已经更新。
```cpp
status_t AudioPolicyManager::setDeviceConnectionStateInt(audio_devices_t device,
                                                         audio_policy_dev_state_t state,
                                                         const char *device_address,
                                                         const char *device_name)
{
    ......
    if (audio_is_output_device(device)) {
        SortedVector <audio_io_handle_t> outputs;
        ssize_t index = mAvailableOutputDevices.indexOf(devDesc);
        // save a copy of the opened output descriptors before any output is opened or closed
        // by checkOutputsForDevice(). This will be needed by checkOutputForAllStrategies()
        mPreviousOutputs = mOutputs;
        switch (state)
        {
        // handle output device connection
        case AUDIO_POLICY_DEVICE_STATE_AVAILABLE: {
            ......
            index = mAvailableOutputDevices.add(devDesc);
            ......
        case AUDIO_POLICY_DEVICE_STATE_UNAVAILABLE: {
            ......
        default:
            ALOGE("setDeviceConnectionState() invalid state: %x", state);
            return BAD_VALUE;
        }
        ......
        mpClientInterface->onAudioPortListUpdate();
        return NO_ERROR;
    }  // end if is output device
    // handle input devices
    if (audio_is_input_device(device)) {
      ......
    } // end if is input device
    ALOGW("setDeviceConnectionState() invalid device: %x", device);
    return BAD_VALUE;
}


void AudioPolicyService::AudioPolicyClient::onAudioPortListUpdate()
{
    mAudioPolicyService->onAudioPortListUpdate();
}

void AudioPolicyService::onAudioPortListUpdate()
{
    mOutputCommandThread->updateAudioPortListCommand();
}
void AudioPolicyService::doOnAudioPortListUpdate()
{
    Mutex::Autolock _l(mNotificationClientsLock);
    for (size_t i = 0; i < mNotificationClients.size(); i++) {
        mNotificationClients.valueAt(i)->onAudioPortListUpdate();
    }
}
void AudioPolicyService::NotificationClient::onAudioPortListUpdate()
{
    if (mAudioPolicyServiceClient != 0 && mAudioPortCallbacksEnabled) {
        mAudioPolicyServiceClient->onAudioPortListUpdate();
    }
}

void AudioPolicyService::AudioCommandThread::updateAudioPortListCommand()
{
    sp<AudioCommand> command = new AudioCommand();
    command->mCommand = UPDATE_AUDIOPORT_LIST;
    ALOGV("AudioCommandThread() adding update audio port list");
    sendCommand(command);
}

void AudioSystem::AudioPolicyServiceClient::onAudioPortListUpdate()
{
    Mutex::Autolock _l(mLock);
    for (size_t i = 0; i < mAudioPortCallbacks.size(); i++) {
        mAudioPortCallbacks[i]->onAudioPortListUpdate();
    }
}
```