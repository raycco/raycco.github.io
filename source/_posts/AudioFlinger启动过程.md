---
title: AudioFlinger启动过程
date: 2017-11-09 15:58:02
tags: Android
categories: Android Audio
---
AudioFlinger以media.audio_flinger为名注册到ServiceManager，是Android Audio系统的一个核心服务，是音频策略的执行者，负责输入输出流设备的管理及音频流数据的处理传输。本文以Android 8.0的代码为基础，记录了其启动过程以及AudioFlinger主要类的关系。其代码位于frameworks/av/services/audioflinger。

## AudioFlinger的启动过程
Android 8.0中与7.0相比，在初始化过程中，主要是实例化了mDevicesFactoryHal和mEffectsFactoryHal，作为HAL进程的客户端与HAL进程交互。
```cpp
AudioFlinger::AudioFlinger()
    //变量初始化
    : BnAudioFlinger(),
      mMediaLogNotifier(new AudioFlinger::MediaLogNotifier()),
      mPrimaryHardwareDev(NULL),
      mAudioHwDevs(NULL),
      mHardwareStatus(AUDIO_HW_IDLE),
      mMasterVolume(1.0f),
      mMasterMute(false),
      // mNextUniqueId(AUDIO_UNIQUE_ID_USE_MAX),
      mMode(AUDIO_MODE_INVALID),
      mBtNrecIsOff(false),
      mIsLowRamDevice(true),
      mIsDeviceTypeKnown(false),
      mGlobalEffectEnableTime(0),
      mSystemReady(false)
{
    ......

    mDevicesFactoryHal = DevicesFactoryHalInterface::create();
    mEffectsFactoryHal = EffectsFactoryHalInterface::create();

    ......

}
```
这里我们主要看一下DevicesFactoryHalInterface关系图。EffectsFactoryHalInterface也是类似的情况。
![](https://i.imgur.com/urJudFp.png)
阅读每个类的代码实现，我们发现只有DevicesFactoryHalHybrid实现了DevicesFactoryHalInterface接口的create()函数，所以这里会创建DevicesFactoryHalHybrid实例，而DevicesFactoryHalHybrid会创建DevicesFactoryHalLocal和DevicesFactoryHalHidl实例。DevicesFactoryHalLocal用于直接加载HAL的lib，是为了兼容8.0之前的版本，而DevicesFactoryHalHidl通过binder通信从hwservicemanager中返回IDevicesFactory实例，通过IDevicesFactory的openDevice函数返回具体的Device，并且用DeviceHalHidl封装返回的Device，这里不再会直接加载HAL的lib，后续和HAL的通信完全通过IDevicesFactory接口，具体在AudioPolicyService加载HW module时会更清楚明白。

第一次初始化还会执行onFirstRef()，创建PatchPanel实例且将AudioFlinger实例传入PatchPanel，设置Audio Mode为AUDIO_MODE_NORMAL,并将自己保存在全局变量gAudioFlinger。
```cpp
void AudioFlinger::onFirstRef()
{
    Mutex::Autolock _l(mLock);

    ......

    mPatchPanel = new PatchPanel(this);

    mMode = AUDIO_MODE_NORMAL;

    gAudioFlinger = this;
}
```
Mutex是互斥类，用于多线程访问同一个资源的时候，保证一次只有一个线程能访问该资源。它的工作原理是某一个线程要访问公共资源的时候先锁定这个Mutex，完成操作之后对Mutex解锁，在此期间如果有其它的线程也要访问公共资源，它就先要去锁Mutex，当它发现Mutex已经被锁住了，那么这个线程就是阻塞在那儿。等Mutex解锁之后所有阻塞在Mutex的线程都会醒来，只有第一个醒来的会抢到Mutex，其它没有抢到的发现自己晚了一步，只能继续阻塞在那儿，等待下次机会。Mutex源码位置/system/core/libutils/include/utils。

为了简化一般的Mutex操作，在class Mutex中定义了一个内部类Autolock，它利用{}作用域实现自动解锁，看一下它的构造函数就知道了。
```cpp
class Autolock {
    public:
        inline explicit Autolock(Mutex& mutex) : mLock(mutex)  { mLock.lock(); }
        inline explicit Autolock(Mutex* mutex) : mLock(*mutex) { mLock.lock(); }
        inline ~Autolock() { mLock.unlock(); }
    private:
        Mutex& mLock;
};
```
我们知道在{}中创建的变量，变开这个大括号时就要销毁，于是就自动调用析构函数了。
## AudioFlinger中放音录音线程
AudioFlinger作为音频的核心服务，主要责任是负责放音与录音，下面我们可以大致看看放音与录音线程的关系，在AudioPolicyServic启动过程中我们会看到这些线程的创建。
![](https://i.imgur.com/FiwGgwV.png)
- ThreadBase：ThreadBase以Thread为基类，而又是PlaybackThread、RecordThread和MmapThread的基类。
- RecordThread：音频录音线程，负责音频的录音，没有子类。
- PlaybackThread：代表放音线程，有两个直接子类MixerThread和DirectOutputThread。
- MixerThread：混音放音线程，有子类DuplicatingThread，负责处理标识为
AUDIO_OUTPUT_FLAG_PRIMARY、AUDIO_OUTPUT_FLAG_FAST、AUDIO_OUTPUT_FLAG_DEEP_BUFFER的音频流，MixerThread 可以把多个音轨的数据混音后再输出。
- DirectOutputThread：直接输出放音线程，有子类OffloadThread，负责处理标识为AUDIO_OUTPUT_FLAG_DIRECT的音频流，这种音频流数据不需要软件混音，直接输出到音频设备即可。
- DuplicatingThread：复制输出放音线程，负责复制音频流数据到其他输出设备，使用场景如主声卡设备、蓝牙耳机设备、USB声卡设备同时输出。
- OffloadThread：硬解回放线程，负责处理标识为AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD的音频流，这种音频流未经软件解码的（一般是MP3、AAC等格式的数据），需要输出到硬件解码器，由硬件解码器解码成PCM 数据。
- MmapThread：这个线程是Android 8.0新加入的，用于低延迟的放音与录音，与AAudio有关系，有MmapPlaybackThread和MmapCaptureThread两个子类。
- MmapPlaybackThread：MMAP放音线程，负责处理标识为AUDIO_OUTPUT_FLAG_MMAP_NOIRQ的音频流。
- MmapcaptureThread：MMAP录音线程，负责处理标识为AUDIO_INPUT_FLAG_MMAP_NOIRQ的音频流。
## AudioFlinger中Tracks
Track：音轨，是AudioFlinger中另一个重要的将角色，下面我们可以看看其关系。
![](https://i.imgur.com/HHP2bZp.png)
对于播放对应Track，OutputTrack，TrackHandle及BnAudioTrack，TrackHandle和BnAudioTrack主要用于和Client端Binder通信，真正代表输出音轨的为Track。同样对应录音音轨的是RecordTrack，RecordHandle及BnAudioRecord，RecordHandle和BnAudioRecord也用于Binder通信，真正录音音轨为RecordTrack。而对于MmapTrack稍微不太样，而是定义通用接口MmapStreamInterface封装MmapThread，再封装MmapTrack，不是通过XXXHandle继承BnXXX，这也许是出于latency上的考虑。
## AudioFlinger中的Streams
我们看到在AudioFlinger中以AudioHwDevice封装HAL的DeviceHalHidl，而DeviceHalHidl封装从HAL返回的具体的Device，这就和HAL层的so文件连接在一起，且AudioHwDevice直接或间接依赖AudioStreamOut和AudioStreanIn这样也和HAL层中stream关联上，后续打开音频输入输出以及打开输入音频流及输出音频流做好准备。具体我们可以在AudioPolicyService启动的时候看的更清楚。
![](https://i.imgur.com/bIsUSZO.png)
AudioFlinger中还有很多其他的知识点，后续学到时再慢慢补上。