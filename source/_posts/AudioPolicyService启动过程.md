---
title: AudioPolicyService启动过程
date: 2017-11-10 20:39:29
tags: Android
categories: Android Audio
---
AudioPolicyService在Audio系统另一个重要的服务，是音频策略的制定者，负责音频设备切换的策略抉择、音量调节策略等。本文基于Android 8.0的代码，记录了AudioPolicyService启动的过程，介绍了其中几个比较关键的点，希望以后自己看到此文能快速回想起AudioPolicyService的启动过程。其代码位于frameworks/av/services/audiopolicy。
## audiopolicy的代码结构
1. service目录是AudioPolicyService、AudioPolicyClient、AudioCommandThread及AudioPolicyEffects的定义及实现。
1. managerdefault目录提供AudioPolicyManager的基本实现。
1. manager目录中是一个工厂类可以根据需要生产所需的AudioPolicyManager，一般各个厂商都会自己实现自己的AudioPolicyManager。
1. engine目录定义了AudioPolicyManagerInterface和AudioPolicyManagerObserver接口，
AudioPolicyManagerInterface由Policy Engine实现，AudioPolicyManagerObserver这个观察者由AudioPolicyManager实现，以供Engine访问。
1. engineconfigure和enginedefault目录是Policy Engine的两种实现，可以根据需要选择其中一种。
1. config目录存放audio policy及音量曲线的config文件。
1. common目录下定义公共代码。audio_policy.conf是旧版本的audio policy config文件。
1. AudioPolicyInterface.h定义了AudioPolicyInterface和AudioPolicyClientInterface接口，AudioPolicyInterface由AuidoPolicyManager实现，特定平台AuidoPolicyManager通过AudioPolicyClientInterface接口的实现者AudioPolicyClient控制音频的输入输出。
![](https://i.imgur.com/kZpNl4i.png)
## AudioPolicyService初始化
从Android 8.0的code发现，基本流程还是和之前一样，分别创建了ApmTone、ApmAudio、ApmOutput三个AudioCommandThread线程，分别用于播放tone音、执行audio命令和执行输出命令，创建AudioPolicyClient，创建AudioPolicyManager以及创建AudioPolicyEffects。但已经完全移除了对旧版本的AUDIO_POLICY_HARDWARE_MODULE_ID的支持（不再加载audio_policy.default.so库得到audio_policy_module模块），完全使用新模式。
```cpp
AudioPolicyService::AudioPolicyService()
    : BnAudioPolicyService(), mpAudioPolicyDev(NULL), mpAudioPolicy(NULL),
      mAudioPolicyManager(NULL), mAudioPolicyClient(NULL), mPhoneState(AUDIO_MODE_INVALID)
{
}

void AudioPolicyService::onFirstRef()
{
    {
        Mutex::Autolock _l(mLock);

        // start tone playback thread
        mTonePlaybackThread = new AudioCommandThread(String8("ApmTone"), this);
        // start audio commands thread
        mAudioCommandThread = new AudioCommandThread(String8("ApmAudio"), this);
        // start output activity command thread
        mOutputCommandThread = new AudioCommandThread(String8("ApmOutput"), this);

        mAudioPolicyClient = new AudioPolicyClient(this);
        mAudioPolicyManager = createAudioPolicyManager(mAudioPolicyClient);
    }
    // load audio processing modules
    sp<AudioPolicyEffects>audioPolicyEffects = new AudioPolicyEffects();
    {
        Mutex::Autolock _l(mLock);
        mAudioPolicyEffects = audioPolicyEffects;
    }
}
```
这些类之间的大致关系如下（AudioPolicyClient和AudioCommandThread都为AudioPolicyService的内部类）：
![](https://i.imgur.com/wNiugU3.png)
AudioPolicyService的初始化大致分为3步：
1.创建三个AudioCommandThread（mTonePlaybackThread，mAudioCommandThread和mOutputCommandThread）
2.初始化AudioPolicyManager
 2.1 加载并解析audio_policy_configuration.xml
 2.2 加载对应的HW module
 2.3 初始化Policy Engine
 2.4 打开输入输出
 2.5 确保所有可用输入输出设备和默认输出设备真正可用
3.初始化AudioPolicyEffects
## AudioCommandThread线程
AudioCommandThread采用异步方式来执行audio command，当需要执行上表中的命令时，首先将命令投递到AudioCommandThread的mAudioCommands命令向量表中，然后通过mWaitWorkCV.signal()唤醒AudioCommandThread线程，被唤醒的AudioCommandThread线程执行完command后，又通过mWaitWorkCV.waitRelative(mLock, waitTime)睡眠等待命令到来。
```cpp
void AudioPolicyService::AudioCommandThread::onFirstRef()
{
    run(mName.string(), ANDROID_PRIORITY_AUDIO);
}

status_t AudioPolicyService::AudioCommandThread::volumeCommand(audio_stream_type_t stream,
                              float volume, audio_io_handle_t output, int delayMs)
{
    sp<AudioCommand> command = new AudioCommand();
    command->mCommand = SET_VOLUME;
    sp<VolumeData> data = new VolumeData();
    data->mStream = stream;
    data->mVolume = volume;
    data->mIO = output;
    command->mParam = data;
    command->mWaitStatus = true;
    return sendCommand(command, delayMs);
}

bool AudioPolicyService::AudioCommandThread::threadLoop()
{
    nsecs_t waitTime = -1;

    mLock.lock();
    while (!exitPending())
    {
        sp<AudioPolicyService> svc;
        while (!mAudioCommands.isEmpty() && !exitPending()) {
            nsecs_t curTime = systemTime();
            // commands are sorted by increasing time stamp: execute them from index 0 and up
            if (mAudioCommands[0]->mTime <= curTime) {
                sp<AudioCommand> command = mAudioCommands[0];
                mAudioCommands.removeAt(0);
                mLastCommand = command;

                switch (command->mCommand) {

                ......

                case SET_VOLUME: {
                    VolumeData *data = (VolumeData *)command->mParam.get();
                    ALOGV("AudioCommandThread() processing set volume stream %d, \
                            volume %f, output %d", data->mStream, data->mVolume, data->mIO);
                    command->mStatus = AudioSystem::setStreamVolume(data->mStream,
                                                                    data->mVolume,
                                                                    data->mIO);
                    }break;

                ......

                }
                
                .....

            } else {
                waitTime = mAudioCommands[0]->mTime - curTime;
                break;
            }
        }

        ......

        // At this stage we have either an empty command queue or the first command in the queue
        // has a finite delay. So unless we are exiting it is safe to wait.
        if (!exitPending()) {
            ALOGV("AudioCommandThread() going to sleep");
            if (waitTime == -1) {
                mWaitWorkCV.wait(mLock);
            } else {
                mWaitWorkCV.waitRelative(mLock, waitTime);
            }
        }
    }
    
    ......

    mLock.unlock();
    return false;
}
```
AudioCommandThread是AudioPolicyService的内部类，AudioCommandThread在内部又定义了AudioCommand及AudioCommandData关系如下。
![](https://i.imgur.com/6IUY98f.png)
## AudioPolicyManager初始化
```cpp
AudioPolicyManager::AudioPolicyManager(AudioPolicyClientInterface *clientInterface)
    :
    mLimitRingtoneVolume(false), mLastVoiceVolume(-1.0f),
    mA2dpSuspended(false),
    mAudioPortGeneration(1),
    mBeaconMuteRefCount(0),
    mBeaconPlayingRefCount(0),
    mBeaconMuted(false),
    mTtsOutputAvailable(false),
    mMasterMono(false),
    mMusicEffectOutput(AUDIO_IO_HANDLE_NONE),
    mHasComputedSoundTriggerSupportsConcurrentCapture(false)
{
    mUidCached = getuid();
    mpClientInterface = clientInterface;

    ...... 
    // audio policy config加载

    // 初始化Policy Engine

    for (size_t i = 0; i < mHwModules.size(); i++) {
        // 加载所有的HW module
        ......
        // 打开音频输入输出
    }

    // 确保所有可用输入输出设备真正可用
    ......

    // 确保默认输出设备真正可用
    ALOGE_IF((mPrimaryOutput == 0), "Failed to open primary output");
    updateDevicesAndOutputs();
}
```
### audio policy config加载
进入AudioPolicyManager构造函数，会首先加载audio policy config文件，对于旧版本使用audio_policy.conf，在代码中定义音量曲线，对于新版本使用audio_policy_configuration.xml。对于Android 8.0使用新版本，这个文件一般会位于/odm/etc或/vendor/etc，同时解析音量曲线xml（audio_policy_volumes.xml和default_volume_tables.xml）,比起之前的硬编码音量曲线，灵活性更好。
```cpp
#ifdef USE_XML_AUDIO_POLICY_CONF
    mVolumeCurves = new VolumeCurvesCollection();
    AudioPolicyConfig config(mHwModules, mAvailableOutputDevices, mAvailableInputDevices,
                             mDefaultOutputDevice, speakerDrcEnabled,
                             static_cast<VolumeCurvesCollection *>(mVolumeCurves));
    if (deserializeAudioPolicyXmlConfig(config) != NO_ERROR) {
#else
    mVolumeCurves = new StreamDescriptorCollection();
    AudioPolicyConfig config(mHwModules, mAvailableOutputDevices, mAvailableInputDevices,
                             mDefaultOutputDevice, speakerDrcEnabled);
    if ((ConfigParsingUtils::loadConfig(AUDIO_POLICY_VENDOR_CONFIG_FILE, config) != NO_ERROR) &&
            (ConfigParsingUtils::loadConfig(AUDIO_POLICY_CONFIG_FILE, config) != NO_ERROR)) {
#endif
        ALOGE("could not load audio policy configuration file, setting defaults");
        config.setDefault();
    }

    // must be done after reading the policy (since conditionned by Speaker Drc Enabling)
    // xml模式时这里是一个空函数无实现
    mVolumeCurves->initializeVolumeCurves(speakerDrcEnabled);
```
当执行完deserializeAudioPolicyXmlConfig，会得到mHwModules对应各个Audio HAL模块、mAvailableOutputDevices可用的输出设备、mAvailableInputDevices可用的输入设备、mDefaultOutputDevice默认输出设备、mVolumeCurves音量曲线及speakerDrcEnabled。

audio_policy_configuration.xml同时定义了多个audio接口(HwModules)，每一个audio接口包含若干routes（通路）、devicesPorts（设备）和mixPorts（音频流），而每个mixPorts又包含多个input和output流，每个input和output流又同时支持多种输入输出模式，每种输入输出模式又支持若干种设备。
mixPorts：listing all output and input streams exposed by the audio HAL.
routes：list of possible connections between input and output devices or between stream and devices.
devicePorts：a list of device descriptors for all input and output devices accessible via this module.This contains both permanently attached devices and removable devices.
![](https://i.imgur.com/eEXkdfk.png)
每个stream type会分为四种device category：DEVICE_CATEGORY_HEADSET，DEVICE_CATEGORY_SPEAKER，DEVICE_CATEGORY_EARPIECE及DEVICE_CATEGORY_EXT_MEDIA来定义音量曲线，定义的形式如下，一样是分段定义，在每一段中定义不同的衰减值以控制音量的大小。
It contains a list of points of this curve expressing the attenuation in Millibels for a given volume index from 0 to 100.
```xml
    <volume stream="AUDIO_STREAM_VOICE_CALL" deviceCategory="DEVICE_CATEGORY_HEADSET">
        <point>0,-4200</point>
        <point>33,-2800</point>
        <point>66,-1400</point>
        <point>100,0</point>
    </volume>
```
### 初始化Policy Engine
这里也算一个观察者模式吧，EngineInstance是单例模式，通过EngineInstance创建AudioPolicyManagerInterface，从而创建Policy Engine，然后设置观察者。
```cpp
// Once policy config has been parsed, retrieve an instance of the engine and initialize it.
    audio_policy::EngineInstance *engineInstance = audio_policy::EngineInstance::getInstance();
    if (!engineInstance) {
        ALOGE("%s:  Could not get an instance of policy engine", __FUNCTION__);
        return;
    }
    // Retrieve the Policy Manager Interface
    mEngine = engineInstance->queryInterface<AudioPolicyManagerInterface>();
    if (mEngine == NULL) {
        ALOGE("%s: Failed to get Policy Engine Interface", __FUNCTION__);
        return;
    }
    mEngine->setObserver(this);
    status_t status = mEngine->initCheck();
    (void) status;
    ALOG_ASSERT(status == NO_ERROR, "Policy engine not initialized(err=%d)", status);
```
其大致关系如下。
![](https://i.imgur.com/PDHsnQK.png)
### 加载HW Module
根据audio policy config加载的HwModule真正加载HAL层的HW module。
```cpp

    // mAvailableOutputDevices and mAvailableInputDevices now contain all attached devices
    // open all output streams needed to access attached devices
    audio_devices_t outputDeviceTypes = mAvailableOutputDevices.types();
    audio_devices_t inputDeviceTypes = mAvailableInputDevices.types() & ~AUDIO_DEVICE_BIT_IN;
    for (size_t i = 0; i < mHwModules.size(); i++) {
        mHwModules[i]->mHandle = mpClientInterface->loadHwModule(mHwModules[i]->getName());
        if (mHwModules[i]->mHandle == 0) {
            continue;
        }
        ......
    }
```
从如上代码看出，会执行mpClientInterface->loadHwModule，即调用AudioPolicyClient的loadHwModule函数，又会转到AudioFlinger的loadHwModule。
```cpp
audio_module_handle_t AudioPolicyService::AudioPolicyClient::loadHwModule(const char *name)
{
    sp<IAudioFlinger> af = AudioSystem::get_audio_flinger();
    if (af == 0) {
        ALOGW("%s: could not get AudioFlinger", __func__);
        return AUDIO_MODULE_HANDLE_NONE;
    }

    return af->loadHwModule(name);
}

audio_module_handle_t AudioFlinger::loadHwModule(const char *name)
{
    if (name == NULL) {
        return AUDIO_MODULE_HANDLE_NONE;
    }
    if (!settingsAllowed()) {
        return AUDIO_MODULE_HANDLE_NONE;
    }
    Mutex::Autolock _l(mLock);
    return loadHwModule_l(name);
}

// loadHwModule_l() must be called with AudioFlinger::mLock held
audio_module_handle_t AudioFlinger::loadHwModule_l(const char *name)
{
    for (size_t i = 0; i < mAudioHwDevs.size(); i++) {
        if (strncmp(mAudioHwDevs.valueAt(i)->moduleName(), name, strlen(name)) == 0) {
            ALOGW("loadHwModule() module %s already loaded", name);
            return mAudioHwDevs.keyAt(i);
        }
    }

    sp<DeviceHalInterface> dev;

    int rc = mDevicesFactoryHal->openDevice(name, &dev);
    if (rc) {
        ALOGE("loadHwModule() error %d loading module %s", rc, name);
        return AUDIO_MODULE_HANDLE_NONE;
    }

    ......

    // Check and cache this HAL's level of support for master mute and master
    // volume.  If this is the first HAL opened, and it supports the get
    // methods, use the initial values provided by the HAL as the current
    // master mute and volume settings.

    ......

    audio_module_handle_t handle = 
          (audio_module_handle_t) nextUniqueId(AUDIO_UNIQUE_ID_USE_MODULE);
    mAudioHwDevs.add(handle, new AudioHwDevice(handle, name, dev, flags));

    ALOGI("loadHwModule() Loaded %s audio interface, handle %d", name, handle);

    return handle;

}
```
mDevicesFactoryHal->openDevice(name, &dev),从前面AudioFlinger的启动，我们知道mDevicesFactoryHal是HAL进程的客户端，对于Android 8.0加载HAL so文件已经移到HAL进程中，不再与audioserver处于同一个进程中。在AudioFlinger中使用AudioHwDevice代表HW Module，AudioHwDevice会封装audio_module_handle_t和DeviceHalInterface，并以audio_module_handle_t为key将其保存在AudioFlinger的mAudioHwDevs中，以供后续查询。
到这里就加载系统的音频接口就加载完了，我们大致可以得出如下结果。
![](https://i.imgur.com/ou4M7xS.png)
### 打开音频输出
这里的输出，即mixPorts中outputs，也就是mHwModules中所有OutputProfile（IOProfile），代表了音频输出流。所以会遍历mHwModules中所有OutputProfile，然后SwAudioOutputDescriptor来描述每一个output，最终保存在以audio_io_handle_t为key的mOutputs中，以供后续查询。不过这里会除了AUDIO_OUTPUT_FLAG_DIRECT，AUDIO_OUTPUT_FLAG_DIRECT的output会在使用的时候打开，不会预先open。在打开输出设备后还会标记可用输出设备的可用情况，以备后续确认可用输出设备真正可用。
```cpp
   // open all output streams needed to access attached devices
   // except for direct output streams that are only opened when they are actually
   // required by an app.
   // This also validates mAvailableOutputDevices list
   for (size_t j = 0; j < mHwModules[i]->mOutputProfiles.size(); j++)
   {
        const sp<IOProfile> outProfile = mHwModules[i]->mOutputProfiles[j];
        if (!outProfile->hasSupportedDevices()) {
            continue;
        }
        if ((outProfile->getFlags() & AUDIO_OUTPUT_FLAG_TTS) != 0) {
            mTtsOutputAvailable = true;
        }
        if ((outProfile->getFlags() & AUDIO_OUTPUT_FLAG_DIRECT) != 0) {
            continue;
        }
        audio_devices_t profileType = outProfile->getSupportedDevicesType();
        if ((profileType & mDefaultOutputDevice->type()) != AUDIO_DEVICE_NONE) {
            profileType = mDefaultOutputDevice->type();
        } else {
            // chose first device present in profile's SupportedDevices also part of
            // outputDeviceTypes
            profileType = outProfile->getSupportedDeviceForType(outputDeviceTypes);
        }
        if ((profileType & outputDeviceTypes) == 0) {
            continue;
        }
        sp<SwAudioOutputDescriptor> outputDesc = 
                          new SwAudioOutputDescriptor(outProfile, mpClientInterface);
        const DeviceVector &supportedDevices = outProfile->getSupportedDevices();
        const DeviceVector &devicesForType = 
                              supportedDevices.getDevicesFromType(profileType);
        String8 address = devicesForType.size() > 0 
                          ? devicesForType.itemAt(0)->mAddress : String8("");
        outputDesc->mDevice = profileType;
        audio_config_t config = AUDIO_CONFIG_INITIALIZER;
        config.sample_rate = outputDesc->mSamplingRate;
        config.channel_mask = outputDesc->mChannelMask;
        config.format = outputDesc->mFormat;
        audio_io_handle_t output = AUDIO_IO_HANDLE_NONE;
        status_t status = mpClientInterface->openOutput(outProfile->getModuleHandle(),
                                                        &output,
                                                        &config,
                                                        &outputDesc->mDevice,
                                                        address,
                                                        &outputDesc->mLatency,
                                                        outputDesc->mFlags);
        if (status != NO_ERROR) {
            ......
        } else {
            outputDesc->mSamplingRate = config.sample_rate;
            outputDesc->mChannelMask = config.channel_mask;
            outputDesc->mFormat = config.format;
            for (size_t k = 0; k  < supportedDevices.size(); k++) {
                 ssize_t index = mAvailableOutputDevices.indexOf(supportedDevices[k]);
                 // give a valid ID to an attached device once confirmed it is reachable
                 if (index >= 0 && !mAvailableOutputDevices[index]->isAttached()) {
                     mAvailableOutputDevices[index]->attach(mHwModules[i]);
                 }
            }
            if (mPrimaryOutput == 0 &&
                        outProfile->getFlags() & AUDIO_OUTPUT_FLAG_PRIMARY) {
                mPrimaryOutput = outputDesc;
            }
            addOutput(output, outputDesc);
            setOutputDevice(outputDesc,
                            outputDesc->mDevice,
                            true,
                            0,
                            NULL,
                            address.string());
        }
   }
```
我们看到会使用到mpClientInterface打开输出，即调用AudioPolicyClient的openOutput，即调用
AudioFlinger的openOutput及openOutput_l。首先调用findSuitableHwDev_l查询合适的AudioHwDevice，即通过audio_module_handle_t在之前加载的mAudioHwDevs中去除对应的AudioHwDevice，调用AudioHwDevice的openOutputStream得到AudioStreamOut，然后根据output flag创建相应的Thread，最后以audio_io_handle_t为key将Thread保存在mPlaybackThreads和mMmapThreads。
```cpp
sp<AudioFlinger::ThreadBase> AudioFlinger::openOutput_l(audio_module_handle_t module,
                                                            audio_io_handle_t *output,
                                                            audio_config_t *config,
                                                            audio_devices_t devices,
                                                            const String8& address,
                                                            audio_output_flags_t flags)
{
    AudioHwDevice *outHwDev = findSuitableHwDev_l(module, devices);
    if (outHwDev == NULL) {
        return 0;
    }
    if (*output == AUDIO_IO_HANDLE_NONE) {
        *output = nextUniqueId(AUDIO_UNIQUE_ID_USE_OUTPUT);
    } else {
        return 0;
    }
    mHardwareStatus = AUDIO_HW_OUTPUT_OPEN;
    // FOR TESTING ONLY:
    ......

    AudioStreamOut *outputStream = NULL;
    status_t status = outHwDev->openOutputStream(
            &outputStream,
            *output,
            devices,
            flags,
            config,
            address.string());
    mHardwareStatus = AUDIO_HW_IDLE;
    if (status == NO_ERROR) {
        if (flags & AUDIO_OUTPUT_FLAG_MMAP_NOIRQ) {
            sp<MmapPlaybackThread> thread =
                    new MmapPlaybackThread(this, *output, outHwDev, outputStream,
                                          devices, AUDIO_DEVICE_NONE, mSystemReady);
            mMmapThreads.add(*output, thread);
            ALOGV("openOutput_l() created mmap playback thread: ID %d thread %p",
                  *output, thread.get());
            return thread;
        } else {
            sp<PlaybackThread> thread;
            if (flags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) {
                thread = new OffloadThread(this, outputStream, *output, devices, mSystemReady);
                ALOGV("openOutput_l() created offload output: ID %d thread %p",
                      *output, thread.get());
            } else if ((flags & AUDIO_OUTPUT_FLAG_DIRECT)
                    || !isValidPcmSinkFormat(config->format)
                    || !isValidPcmSinkChannelMask(config->channel_mask)) {
                thread = new DirectOutputThread(this, outputStream, 
                                         *output, devices, mSystemReady);
                ALOGV("openOutput_l() created direct output: ID %d thread %p",
                      *output, thread.get());
            } else {
                thread = new MixerThread(this, outputStream, *output, devices, mSystemReady);
                ALOGV("openOutput_l() created mixer output: ID %d thread %p",
                      *output, thread.get());
            }
            mPlaybackThreads.add(*output, thread);
            return thread;
        }
    }
    return 0;
}
status_t AudioFlinger::openOutput(audio_module_handle_t module,
                                  audio_io_handle_t *output,
                                  audio_config_t *config,
                                  audio_devices_t *devices,
                                  const String8& address,
                                  uint32_t *latencyMs,
                                  audio_output_flags_t flags)
{
    if (devices == NULL || *devices == AUDIO_DEVICE_NONE) {
        return BAD_VALUE;
    }
    Mutex::Autolock _l(mLock);
    sp<ThreadBase> thread = openOutput_l(module, output, config, *devices, address, flags);
    if (thread != 0) {
        if ((flags & AUDIO_OUTPUT_FLAG_MMAP_NOIRQ) == 0) {
            PlaybackThread *playbackThread = (PlaybackThread *)thread.get();
            *latencyMs = playbackThread->latency();
            // notify client processes of the new output creation
            playbackThread->ioConfigChanged(AUDIO_OUTPUT_OPENED);
            // the first primary output opened designates the primary hw device
            if ((mPrimaryHardwareDev == NULL) && (flags & AUDIO_OUTPUT_FLAG_PRIMARY)) {
                ALOGI("Using module %d as the primary audio interface", module);
                mPrimaryHardwareDev = playbackThread->getOutput()->audioHwDev;
                AutoMutex lock(mHardwareLock);
                mHardwareStatus = AUDIO_HW_SET_MODE;
                mPrimaryHardwareDev->hwDevice()->setMode(mMode);
                mHardwareStatus = AUDIO_HW_IDLE;
            }
        } else {
            MmapThread *mmapThread = (MmapThread *)thread.get();
            mmapThread->ioConfigChanged(AUDIO_OUTPUT_OPENED);
        }
        return NO_ERROR;
    }
    return NO_INIT;
}
```
我们可以看到在AudioPolicyManager中有mOutputs以audio_io_handle_t为key保存SwAudioOutputDescriptor，而在AudioFlinger中mPlaybackThreads和mMmapThreads以audio_io_handle_t保存线程，所以我们可以得出如下关系图。
![](https://i.imgur.com/K5J8Y6B.png)
### 打开音频输入
打开音频输入和打开音频输出很类似，只是将SwAudioOutputDescriptor、PlaybackThread及AudioStreamOut换成了AudioInputDescriptor、RecordThread及AudioStreamIn，这里就一笔带过。
### 确保可用输入输出和默认输出真正可用
无其他。主要是移除不可达设备。
```cpp
    // make sure all attached devices have been allocated a unique ID
    // 确保所有可用输出设备真正可用
    for (size_t i = 0; i  < mAvailableOutputDevices.size();) {
        if (!mAvailableOutputDevices[i]->isAttached()) {
            ALOGW("Output device %08x unreachable", mAvailableOutputDevices[i]->type());
            mAvailableOutputDevices.remove(mAvailableOutputDevices[i]);
            continue;
        }
        // The device is now validated and can be appended to the available 
        // devices of the engine
        // 目前不做任何处理
        mEngine->setDeviceConnectionState(mAvailableOutputDevices[i],
                                          AUDIO_POLICY_DEVICE_STATE_AVAILABLE);
        i++;
    }
    // 确保所有可用输入设备真正可用
    for (size_t i = 0; i  < mAvailableInputDevices.size();) {
        if (!mAvailableInputDevices[i]->isAttached()) {
            ALOGW("Input device %08x unreachable", mAvailableInputDevices[i]->type());
            mAvailableInputDevices.remove(mAvailableInputDevices[i]);
            continue;
        }
        // The device is now validated and can be appended to the available devices of the engine
        // 目前不做任何处理
        mEngine->setDeviceConnectionState(mAvailableInputDevices[i],
                                          AUDIO_POLICY_DEVICE_STATE_AVAILABLE);
        i++;
    }

    // make sure default device is reachable
    // 确保默认输出设备真正可用
    if (mDefaultOutputDevice == 0 || 
        mAvailableOutputDevices.indexOf(mDefaultOutputDevice) < 0) {
        ALOGE("Default device %08x is unreachable", mDefaultOutputDevice->type());
    }
    
    ALOGE_IF((mPrimaryOutput == 0), "Failed to open primary output");
    updateDevicesAndOutputs();
```
## AudioPolicyEffects初始化
对于音效策略，类似会先加载audio_effects.conf，这个文件可能位于system/etc/或者vendor/etc/。
```cpp
AudioPolicyEffects::AudioPolicyEffects()
{
    // load automatic audio effect modules
    if (access(AUDIO_EFFECT_VENDOR_CONFIG_FILE, R_OK) == 0) {
        loadAudioEffectConfig(AUDIO_EFFECT_VENDOR_CONFIG_FILE);
    } else if (access(AUDIO_EFFECT_DEFAULT_CONFIG_FILE, R_OK) == 0) {
        loadAudioEffectConfig(AUDIO_EFFECT_DEFAULT_CONFIG_FILE);
    }
}
```
这个文件的格式如下。
```cpp
# List of effect libraries to load. Each library element must contain a "path" element
# giving the full path of the library .so file.
#    libraries {
#        <lib name> {
#          path <lib path>
#        }
#    }
# list of effects to load. Each effect element must contain a "library" and a "uuid" element.
# The value of the "library" element must correspond to the name of one library element in the
# "libraries" element.
# The name of the effect element is indicative, only the value of the "uuid" element
# designates the effect.
# The uuid is the implementation specific UUID as specified by the effect vendor. This is not the
# generic effect type UUID.
#    effects {
#        <fx name> {
#            library <lib name>
#            uuid <effect uuid>
#        }
#        ...
#    }
```
到这里AudioPolicyService的启动流程已经完结，且篇幅已经挺长，其他的知识点，学习到时再补上。