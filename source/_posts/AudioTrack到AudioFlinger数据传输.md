---
title: AudioTrack到AudioFlinger数据传输：共享内存
date: 2017-12-28 15:49:05
tags: Android
categories: Android Audio
---
从前面AudioFlinger启动知道，其运行在audioserver进程，而创建AudioTrack一般在APP端创建，APP一般运行在单独的进程，所以这里说AudioTrack到AudioFlinger数据传输，其实也是两个进程之间数据传输，这篇笔记就记录了AudioTrack是怎么将Audio数据转移到AudioFlinger的。
## 通信模型
在AudioTrack和AudioFlinger通信过程中，有两种情形：MODE_STATIC和MODE_STREAM。

MODE_STATIC主要针对数据量较小及延迟要求高的音频，比如短促的游戏音乐，这种情况下，共享内存的创建在Client进程，且一般也不会动态处理这段buffer，AudioTrack会一次性将数据通过共享内存传递到AudioFlinger。

MODE_STREAM适用于大多数应用场景，是Android主要的音频播放方式，如下图，就是MODE_STREAM模式下，AudioTrack到AudioFlinger数据传输的模式，同样使用共享内存，且是环形buffer，通过生产者-消费者模式进行数据传输。与MODE_STATIC的主要区别在于共享内存的创建方式及对共享内存的控制方式不太一样。
![](https://i.imgur.com/uFXUT2x.png)
## 共享内存控制块
对于共享内存的通信方式，涉及到两个进程对于共享内存的读写，这就会有进程间的同步，所以共享内存控制块起着关键性的作用，所以下面我们看看其定义：frameworks/av/include/private/media/AudioTrackShared.h
```cpp
// for audio_track_cblk_t::mFlags
#define CBLK_UNDERRUN   0x01 // set by server immediately on output underrun, cleared by client
#define CBLK_FORCEREADY 0x02 // set: track is considered ready immediately by AudioFlinger,
                             // clear: track is ready when buffer full
#define CBLK_INVALID    0x04 // track buffer invalidated by AudioFlinger, need to re-create
#define CBLK_DISABLED   0x08 // output track disabled by AudioFlinger due to underrun,
                             // need to re-start.  Unlike CBLK_UNDERRUN, this is not set
                             // immediately, but only after a long string of underruns.
// 0x10 unused
#define CBLK_LOOP_CYCLE 0x20 // set by server each time a loop 
                             // cycle other than final one completes
#define CBLK_LOOP_FINAL 0x40 // set by server when the final loop cycle completes
#define CBLK_BUFFER_END 0x80 // set by server when the position 
                             // reaches end of buffer if not looping
#define CBLK_OVERRUN   0x100 // set by server immediately on input overrun, cleared by client
#define CBLK_INTERRUPT 0x200 // set by client on interrupt(), cleared by client in obtainBuffer()
#define CBLK_STREAM_END_DONE 0x400 // set by server on render completion, cleared by client
```
如上是对于传输的状态的定义。
- CBLK_UNDERRUN：AudioTrack写入数据的速度跟不上AudioFlinger读取数据的速度，使得 AudioFlinger不能及时获取到预期的数据量，反映到现实的后果就是声音断续；这种情况的根本原因大多是应用程序不能及时写入数据或者缓冲区分配过小，AudioTrack本身并没有错；AudioFlinger针对这点做了容错处理：当发现underrun时，先陷入短时间的睡眠，不急着读取数据，让应用程序准备更多的数据。
- CBLK_OVERRUN：刚好和CBLK_UNDERRUN相反，主要对于AudioRecord，底层写数据的速度太快，AudioRecord读取数据比较慢，数据传输就会处于超负荷的运行。

```cpp
// Important: do not add any virtual methods, including ~
struct audio_track_cblk_t
{
                // Since the control block is always located in shared memory, this constructor
                // is only used for placement new(). It is never used for regular new() or stack.
                            audio_track_cblk_t();
                /*virtual*/ ~audio_track_cblk_t() { }
                ......               // friend classes of Proxy
    // The data members are grouped so that members accessed frequently and in the same context
    // are in the same line of data cache.
                uint32_t    mServer; // Number of filled frames consumed by server (mIsOut),
                                     // or filled frames provided by server (!mIsOut).
                                     // It is updated asynchronously by server without 
                                     // a barrier. The value should be used
                                     // "for entertainment purposes only",
                                     // which means don't make important decisions based on it.
    volatile    int32_t     mFutex;  // event flag: down (P) by client,
                                     // up (V) by server or binderDied() or interrupt()
#define CBLK_FUTEX_WAKE 1        // if event flag bit is set, then a deferred wake is pending
private:
                // This field should be a size_t, but since it is located in shared 
                // memory we force to 32-bit. The client and server may have 
                // different typedefs for size_t.
                uint32_t    mMinimum;    // server wakes up client if available >= mMinimum
                // Stereo gains for AudioTrack only, not used by AudioRecord.
                gain_minifloat_packed_t mVolumeLR;
                uint32_t    mSampleRate; // AudioTrack only: client's requested 
                                         // sample rate in Hz or 0 == default. 
                                         // Write-only client, read-only server.
                PlaybackRateQueue::Shared mPlaybackRateQueue;
                // client write-only, server read-only
                uint16_t    mSendLevel;      // Fixed point U4.12 so 0x1000 means 1.0
                // server write-only, client read
                ExtendedTimestampQueue::Shared mExtendedTimestampQueue;
                // This is set by AudioTrack.setBufferSizeInFrames().
                // A write will not fill the buffer above this limit.
    volatile    uint32_t   mBufferSizeInFrames;  // effective size of the buffer
public:
    volatile    int32_t     mFlags;         // combinations of CBLK_*
public:
                union {
                    AudioTrackSharedStreaming   mStreaming;
                    AudioTrackSharedStatic      mStatic;
                    int                         mAlign[8];
                } u;
                // Cache line boundary (32 bytes)
};
```
我们比较关注的mFutex，用于进程的同步，mMinimum唤醒server最小值，mFlags标记当前传输处于什么样的状态，没有直接看到环形buffer相关控制变量，但是有一个联合体。

对于MODE_STREAM：
```cpp
struct AudioTrackSharedStreaming {
    // similar to NBAIO MonoPipe
    // in continuously incrementing frame units, take modulo buffer size, 
    // which must be a power of 2
    volatile int32_t mFront;    // read by consumer (output: server, input: client)
    volatile int32_t mRear;     // written by producer (output: client, input: server)
    volatile int32_t mFlush;    // incremented by client to indicate a request to flush;
                                // server notices and discards all data between mFront and mRear
    volatile uint32_t mUnderrunFrames; // server increments for each unavailable 
                                       // but desired frame
    volatile uint32_t mUnderrunCount;  // server increments for each underrun occurrence
};
```
## 共享内存的创建
### 时序图
首先通过时序图看看匿名共享内存的创建过程，对于MODE_STREAM，匿名共享内创建在AudioFlinger中，即在audioserver中，其中会通过Binder通信将创建好的共享内存随Track的返回而返回到AudioTrack的client中，通过获取共享内存地址再重新映射到client进程，这样两个进程就可以操作这块内存传输数据了。
![](https://i.imgur.com/azF90j1.png)
### 源代码
看完时序图，大致看看源代码，看看其创建过程。
```cpp
status_t AudioTrack::set(
        ......
        const sp<IMemory>& sharedBuffer,
        ......)
{
    ......
    mSharedBuffer = sharedBuffer;
    ......
    // create the IAudioTrack
    status_t status = createTrack_l();
    ......
}
```
在创建AudioTrack的时候，有一个比较重要的参数sharedBuffer，这个参数决定了创建共享内存的进程，若为空，则会在AudioFlinger创建，否则在创建AudioTrack之前就创建。
```cpp
status_t AudioTrack::createTrack_l()
{
    const sp<IAudioFlinger>& audioFlinger = AudioSystem::get_audio_flinger();
    ......
    audio_io_handle_t output;
    audio_stream_type_t streamType = mStreamType;
    ......
    status = AudioSystem::getOutputForAttr(attr, &output,
                                           mSessionId, &streamType, mClientUid,
                                           &config,
                                           mFlags, mSelectedDeviceId, &mPortId);
    ......
    sp<IAudioTrack> track = audioFlinger->createTrack(streamType,
                                                      mSampleRate,
                                                      mFormat,
                                                      mChannelMask,
                                                      &temp,
                                                      &flags,
                                                      mSharedBuffer,
                                                      output,
                                                      mClientPid,
                                                      tid,
                                                      &mSessionId,
                                                      mClientUid,
                                                      &status,
                                                      mPortId);
    ......
    sp<IMemory> iMem = track->getCblk(); // 这里获取整个匿名共享内存IMemory
    ......
    void *iMemPointer = iMem->pointer(); // 映射匿名共享内存到当前进程，获取共享内存首地址
    ......
    mAudioTrack = track; // 保存AudioFlinger::PlaybackThread::Track的代理对象 IAudioTrack
    mCblkMemory = iMem;  // 保存匿名共享内存

    // 控制块位于匿名共享内存的首部
    audio_track_cblk_t* cblk = static_cast<audio_track_cblk_t*>(iMemPointer);
    mCblk = cblk;
    ......
    void* buffers;
    if (mSharedBuffer == 0) {
        buffers = cblk + 1;
    } else {
        buffers = mSharedBuffer->pointer();
        if (buffers == NULL) {
            ALOGE("Could not get buffer pointer");
            return NO_INIT;
        }
    }
    ......
    mServer = 0;
    // update proxy
    if (mSharedBuffer == 0) {
        mStaticProxy.clear();
        // 当mSharedBuffer为空，意味着音轨数据模式为MODE_STREAM，那么创建
        // AudioTrackClientProxy对象
        mProxy = new AudioTrackClientProxy(cblk, buffers, frameCount, mFrameSize);
    } else {
        // 当mSharedBuffer非空，意味着音轨数据模式为MODE_STATIC，那么创建
        // StaticAudioTrackClientProxy对象
        mStaticProxy = new StaticAudioTrackClientProxy(cblk, buffers, frameCount, mFrameSize);
        mProxy = mStaticProxy;
    }
    ......
}
```
在AudioPolicyService启动的时候在AudioPolicyService中会保存着与放音线程PlaybackThread对应的output，这里首先通过streamType等参数获取audio_io_handle_t代表的output，然后进入AudioFlinger创建IAudioTrack。通过IAudioTrack获取整个匿名共享内存IMemory，然后再映射匿名共享内存到当前进程，保存到mCblkMemory，mCblkMemory的首部是匿名共享内存控制块audio_track_cblk_t，最后创建AudioTrackClientProxy，后续通过AudioTrackClientProxy操作共享内存写数据。
```cpp
// frameworks/av/services/audioflinger/AudioFlinger.cpp
sp<IAudioTrack> AudioFlinger::createTrack(
        ......
        const sp<IMemory>& sharedBuffer,
        audio_io_handle_t output,
        ......)
{
    sp<PlaybackThread::Track> track;
    sp<TrackHandle> trackHandle;
    sp<Client> client;
    ......
    {
        Mutex::Autolock _l(mLock);
        // 根据传入来的audio_io_handle_t，找到对应的PlaybackThread
        PlaybackThread *thread = checkPlaybackThread_l(output);
        if (thread == NULL) {
            ALOGE("no playback thread found for output handle %d", output);
            lStatus = BAD_VALUE;
            goto Exit;
        }
        client = registerPid(pid);
        ......
        track = thread->createTrack_l(client, streamType, sampleRate, format,
                channelMask, frameCount, sharedBuffer, lSessionId, flags, tid,
                clientUid, &lStatus, portId);
        ......
    }
    // return handle to client
    trackHandle = new TrackHandle(track);// 创建Track的代理TrackHandle并返回
Exit:
    *status = lStatus;
    return trackHandle;
}
```
在AudioFlinger中首先根据audio_io_handle_t找到相应的放音线程PlaybackThread，创建Client保存对应的客户端，在PlaybackThread中创建Track，最后创建Track的代理TrackHandle并返回到AudioTrack。
```cpp
// frameworks/av/services/audioflinger/Threads.cpp
sp<AudioFlinger::PlaybackThread::Track> AudioFlinger::PlaybackThread::createTrack_l(
        const sp<AudioFlinger::Client>& client,
        ......
        const sp<IMemory>& sharedBuffer,
        ......)
{
    size_t frameCount = *pFrameCount;
    sp<Track> track;
    status_t lStatus;
    ......
    // For normal PCM streaming tracks, update minimum frame count.
    // For compatibility with AudioTrack calculation, buffer depth is forced
    // to be at least 2 x the normal mixer frame count and cover audio hardware latency.
    // This is probably too conservative, but legacy application code may depend on it.
    // If you change this calculation, also review the start threshold which is related.
    if (!(*flags & AUDIO_OUTPUT_FLAG_FAST)
            && audio_has_proportional_frames(format) && sharedBuffer == 0) {
        // this must match AudioTrack.cpp calculateMinFrameCount().
        // TODO: Move to a common library
        uint32_t latencyMs = 0;
        lStatus = mOutput->stream->getLatency(&latencyMs);
        if (lStatus != OK) {
            ALOGE("Error when retrieving output stream latency: %d", lStatus);
            goto Exit;
        }
        uint32_t minBufCount = latencyMs / ((1000 * mNormalFrameCount) / mSampleRate);
        if (minBufCount < 2) {
            minBufCount = 2;
        }
        // For normal mixing tracks, if speed is > 1.0f (normal), AudioTrack
        // or the client should compute and pass in a larger buffer request.
        size_t minFrameCount =
                minBufCount * sourceFramesNeededWithTimestretch(
                        sampleRate, mNormalFrameCount,
                        mSampleRate, AUDIO_TIMESTRETCH_SPEED_NORMAL /*speed*/);
        if (frameCount < minFrameCount) { // including frameCount == 0
            frameCount = minFrameCount;
        }
    }
    ......
    { // scope for mLock
        Mutex::Autolock _l(mLock);
        ......
        track = new Track(this, client, streamType, sampleRate, format,
                          channelMask, frameCount, NULL, sharedBuffer,
                          sessionId, uid, *flags, TrackBase::TYPE_DEFAULT, portId);
        ......
    }
    lStatus = NO_ERROR;
Exit:
    *status = lStatus;
    return track;
}
```
根据HAL的参数获取延迟latencyMs，mNormalFrameCount是在放音线程PlaybackThread创建时，预先初始化的frame count，根据这两个值得出播放当前音频需要的最小的buffer count，继而的到最小的frame count，然后创建Track。
```cpp
// frameworks/av/services/audioflinger/Tracks.cpp
AudioFlinger::PlaybackThread::Track::Track(
            PlaybackThread *thread,
            const sp<Client>& client,
            audio_stream_type_t streamType,
            uint32_t sampleRate,
            audio_format_t format,
            audio_channel_mask_t channelMask,
            size_t frameCount,
            void *buffer,
            const sp<IMemory>& sharedBuffer,
            audio_session_t sessionId,
            uid_t uid,
            audio_output_flags_t flags,
            track_type type,
            audio_port_handle_t portId)
    :   TrackBase(thread, client, sampleRate, format, channelMask, frameCount,
                  (sharedBuffer != 0) ? sharedBuffer->pointer() : buffer,
                  sessionId, uid, true /*isOut*/, (type == TYPE_PATCH) ? 
                  ( buffer == NULL ? ALLOC_LOCAL : ALLOC_NONE) : ALLOC_CBLK,
                  type, portId),
    mFillingUpStatus(FS_INVALID),
    // mRetryCount initialized later when needed
    mSharedBuffer(sharedBuffer),
    ......
{
    ......
    if (sharedBuffer == 0) {
        // 数据传输模式为MODE_STREAM模式，创建一个AudioTrackServerProxy对象
        // PlaybackThread将持续使用它从环形buffer上取得可读数据的位置
        mAudioTrackServerProxy = new AudioTrackServerProxy(mCblk, mBuffer, frameCount,
                mFrameSize, !isExternalTrack(), sampleRate);
    } else {
        // Is the shared buffer of sufficient size?
        // (frameCount * mFrameSize) is <= SIZE_MAX, checked in TrackBase.
        if (sharedBuffer->size() < frameCount * mFrameSize) {
            // Workaround: clear out mCblk to indicate track hasn't been properly created.
            mCblk->~audio_track_cblk_t();   // destroy our shared-structure.
            if (mClient == 0) {
                free(mCblk);
            }
            mCblk = NULL;
            mSharedBuffer.clear(); // release shared buffer early
            android_errorWriteLog(0x534e4554, "38340117");
            return;
        }
        // 数据传输模式为MODE_STATIC模式，创建一个StaticAudioTrackServerProxy对象
        mAudioTrackServerProxy = new StaticAudioTrackServerProxy(mCblk, mBuffer, frameCount,
                mFrameSize);
    }
    mServerProxy = mAudioTrackServerProxy;
    ......
}

// frameworks/av/services/audioflinger/Tracks.cpp
AudioFlinger::ThreadBase::TrackBase::TrackBase(
            ThreadBase *thread,
            const sp<Client>& client,
            uint32_t sampleRate,
            audio_format_t format,
            audio_channel_mask_t channelMask,
            size_t frameCount,
            void *buffer,
            audio_session_t sessionId,
            uid_t clientUid,
            bool isOut,
            alloc_type alloc,
            track_type type,
            audio_port_handle_t portId)
    :   RefBase(),
        mThread(thread),
        mClient(client),
        mFrameSize(audio_has_proportional_frames(format) ?
                mChannelCount * audio_bytes_per_sample(format) : sizeof(int8_t))
        ......
{
    ......
    size_t bufferSize = buffer == NULL ? roundup(frameCount) : frameCount;
    bufferSize *= mFrameSize;
    size_t size = sizeof(audio_track_cblk_t);
    if (buffer == NULL && alloc == ALLOC_CBLK) {
        // check overflow when computing allocation size for streaming tracks.
        if (size > SIZE_MAX - bufferSize) {
            android_errorWriteLog(0x534e4554, "34749571");
            return;
        }
        size += bufferSize;
    }
    if (client != 0) {
        mCblkMemory = client->heap()->allocate(size); // 创建共享内存并映射当前进程
        if (mCblkMemory == 0 ||
                (mCblk = static_cast<audio_track_cblk_t *>(mCblkMemory->pointer())) == NULL) {
            ALOGE("not enough memory for AudioTrack size=%zu", size);
            client->heap()->dump("AudioTrack");
            mCblkMemory.clear();
            return;
        }
    } 
    ......
    // construct the shared structure in-place.
    if (mCblk != NULL) {
        // 这是C++的placement new（定位创建对象）语法：new(@BUFFER) @CLASS();
        // 可以在特定内存位置上构造一个对象
        // 这里，在匿名共享内存首地址上构造了一个audio_track_cblk_ 对象
        // 这样AudioTrack与AudioFlinger都能访问这个audio_track_cblk_t对象了
        new(mCblk) audio_track_cblk_t();
        switch (alloc) {
        ......
        case ALLOC_CBLK:
            // clear all buffers
            if (buffer == NULL) {
                // 数据传输模式为MODE_STREAM时，数据buffer的分配
                // 数据buffer的首地址紧靠控制块（audio_track_cblk_t）之后
                mBuffer = (char*)mCblk + sizeof(audio_track_cblk_t);
                memset(mBuffer, 0, bufferSize);
            } else {
                // 数据传输模式为MODE_STATIC时，直接指向sharedBuffer
                // sharedBuffer是应用进程分配的匿名共享内存，应用进程已经一次性把数据
                // 写到sharedBuffer来了，AudioFlinger可以直接从这里读取
                mBuffer = buffer;
            }
            break;
        ......
        }
    }
}

// system/meida/audio/include/system/audio.h
static inline size_t audio_bytes_per_sample(audio_format_t format)
{
    size_t size = 0;

    switch (format) {
    case AUDIO_FORMAT_PCM_32_BIT:
    case AUDIO_FORMAT_PCM_8_24_BIT:
        size = sizeof(int32_t);
        break;
    case AUDIO_FORMAT_PCM_24_BIT_PACKED:
        size = sizeof(uint8_t) * 3;
        break;
    case AUDIO_FORMAT_PCM_16_BIT:
    case AUDIO_FORMAT_IEC61937:
        size = sizeof(int16_t);
        break;
    case AUDIO_FORMAT_PCM_8_BIT:
        size = sizeof(uint8_t);
        break;
    case AUDIO_FORMAT_PCM_FLOAT:
        size = sizeof(float);
        break;
    default:
        break;
    }
    return size;
}
```
Track从TrackBase继承而来，所以会先执行TrackBase构造函数，mFrameSize会根据format和channel count计算而得，比如双声道，forma为AUDIO_FORMAT_PCM_16_BIT，则mFrameSize为4Byte。计算共享内存大小，其大小为控制块大小加上真正数据大小。
size = sizeof(audio_track_cblk_t) + framecount * mFrameSize
然后会创建匿名共享内存并且映射到当前进程mCblkMemory = client->heap()->allocate(size);
然后使用C++的placement new（定位创建对象）在匿名共享内存上创建共享内存控制块。最后将mBuffer清空，创建AudioTrackServerProxy，后续一次操作共享内存读取数据。
```cpp
sp<AudioFlinger::Client> AudioFlinger::registerPid(pid_t pid)
{
    Mutex::Autolock _cl(mClientLock);
    // If pid is already in the mClients wp<> map, then use that entry
    // (for which promote() is always != 0), otherwise create a new entry and Client.
    sp<Client> client = mClients.valueFor(pid).promote();
    if (client == 0) {
        client = new Client(this, pid);
        mClients.add(pid, client);
    }
    return client;
}

// Max shared memory size for audio tracks and audio records per client process
static const size_t kClientSharedHeapSizeBytes = 1024*1024;
// Shared memory size multiplier for non low ram devices
static const size_t kClientSharedHeapSizeMultiplier = 4;

AudioFlinger::Client::Client(const sp<AudioFlinger>& audioFlinger, pid_t pid)
    :   RefBase(),
        mAudioFlinger(audioFlinger),
        mPid(pid)
{
    size_t heapSize = property_get_int32("ro.af.client_heap_size_kbyte", 0);
    heapSize *= 1024;
    if (!heapSize) {
        heapSize = kClientSharedHeapSizeBytes;
        // Increase heap size on non low ram devices to limit risk of reconnection 
        // failure for invalidated tracks
        // 目前一般的设备都不是RAM比较小的设备，所以一般默认申请4M内存
        if (!audioFlinger->isLowRamDevice()) {
            heapSize *= kClientSharedHeapSizeMultiplier;
        }
    }
    mMemoryDealer = new MemoryDealer(heapSize, "AudioFlinger::Client");
}
```
创建共享内存使用了Client类，client->heap()->allocate(size)，创建Client即创建内存管理接口MemoryDealer，即MemoryHeapBase。下一节分析匿名共享内存的C++接口，再具体分析匿名共享内存的管理。
## 匿名共享内存C++接口
### MemoryHeapBase
MemoryHeapBase类的对象可以作为Binder对象在进程间传输，作为一个Binder对象，就有Server端对象和Client端引用的概念。下面先看Server端的实现。
![](https://i.imgur.com/fWwABNR.png)
共同基类RefBase，主要用于智能指针的使用。在ProcessState中打开Binder驱动，IPCThreadState通过ProcessState和Binder驱动交互，当有服务请求时，会先调用IPCThreadState的transact函数，进而调用BBinder的transact函数，根据继承关系会调动BnMemoryHeap的onTransact函数，根据code决定调用具体实现函数，从而进入MemoryHeapBase。这就是一次通信在服务端的运行流程。

MemoryHeapBase要创建共享内存，接下来看看其中一个构造函数具体实现。
```cpp
// frameworks/native/libs/binder/MemoryHeapBase.cpp
MemoryHeapBase::MemoryHeapBase(size_t size, uint32_t flags, char const * name)
    : mFD(-1), mSize(0), mBase(MAP_FAILED), mFlags(flags),
      mDevice(0), mNeedUnmap(false), mOffset(0)
{
    // 获得系统中一页大小的内存
    const size_t pagesize = getpagesize();
    // 内存页对齐
    size = ((size + pagesize-1) & ~(pagesize-1));
    // 创建一块匿名共享内存
    int fd = ashmem_create_region(name == NULL ? "MemoryHeapBase" : name, size);
    ALOGE_IF(fd<0, "error creating ashmem region: %s", strerror(errno));
    if (fd >= 0) {
        // 将创建的匿名共享内存映射到当前进程地址空间中
        if (mapfd(fd, size) == NO_ERROR) {
            // 如果地址映射成功，修改匿名共享内存的访问属性
            if (flags & READ_ONLY) {
                ashmem_set_prot_region(fd, PROT_READ);
            }
        }
    }
}
```
在以上构造函数中根据参数，利用匿名共享内存提供的C语言接口创建一块匿名共享内存，并映射到当前进程的虚拟地址空间，参数size是指定匿名共享内存的大小，flags指定匿名共享内存的访问属性，name指定匿名共享内存的名称，如果没有指定名称，默认命名为MemoryHeapBase。
```cpp
// frameworks/native/libs/binder/MemoryHeapBase.cpp
status_t MemoryHeapBase::mapfd(int fd, size_t size, uint32_t offset)
{
    if (size == 0) {
        // try to figure out the size automatically
        struct stat sb;
        if (fstat(fd, &sb) == 0)
            size = sb.st_size;
        // if it didn't work, let mmap() fail.
    }
    if ((mFlags & DONT_MAP_LOCALLY) == 0) {
        // 通过mmap系统调用进入内核空间的匿名共享内存驱动，
        // 并调用ashmem_mmap函数将匿名共享内存映射到当前进程
        void* base = (uint8_t*)mmap(0, size,
                PROT_READ|PROT_WRITE, MAP_SHARED, fd, offset);
        if (base == MAP_FAILED) {
            ALOGE("mmap(fd=%d, size=%u) failed (%s)",
                    fd, uint32_t(size), strerror(errno));
            close(fd);
            return -errno;
        }
        //ALOGD("mmap(fd=%d, base=%p, size=%lu)", fd, base, size);
        mBase = base;
        mNeedUnmap = true;
    } else  {
        mBase = 0; // not MAP_FAILED
        mNeedUnmap = false;
    }
    mFD = fd;
    mSize = size;
    mOffset = offset;
    return NO_ERROR;
}
```
mmap函数的第一个参数0表示由内核来决定这个匿名共享内存文件在进程地址空间的起始位置，第二个参数size表示要映射的匿名共享内文件的大小，第三个参数PROT_READ|PROT_WRITE表示这个匿名共享内存是可读写的，第四个参数fd指定要映射的匿名共享内存的文件描述符，第五个参数offset表示要从这个文件的哪个偏移位置开始映射。调用了这个函数之后，最后会进入到内核空间的ashmem驱动程序模块中去执行ashmem_mmap函数，调用mmap函数返回之后，就得这块匿名共享内存在本进程地址空间中的起始访问地址了，将这个地址保存在成员变量mBase中，最后，还将这个匿名共享内存的文件描述符和以及大小分别保存在成员变量mFD和mSize，并提供了相应接口函数来访问这些变量值。通过构造MemoryHeapBase对象就可以创建一块匿名共享内存，或者映射一块已经创建的匿名共享内存到当前进程的地址空间。

接下来我们再来看一下MemoryHeapBase在Client端实现。
![](https://i.imgur.com/RATRreG.png)
在和匿名共享内存操作相关的类中，BpMemoryHeap类是前面分析的MemoryHeapBase类在Client端进程的远接接口类，当Client端进程从Service Manager或者其它途径获得了一个MemoryHeapBase对象的引用之后，就会在本地创建一个BpMemoryHeap对象来代表这个引用。BpMemoryHeap类同样是要实现IMemoryHeap接口，Client端将调用传递给BpMemoryHeap，BpMemoryHeap的基类BpRefBase有一个mRemote对象，指向BpBinder,所以通过调用BpBinder的transact函数进而调用IPCThreadState，将请求传递给Server端。

BpMemoryHeap的声明如下：
```cpp
// frameworks/native/libs/binder/IMemory.cpp
class BpMemoryHeap : public BpInterface<IMemoryHeap>
{
public:
    explicit BpMemoryHeap(const sp<IBinder>& impl);
    virtual ~BpMemoryHeap();
    virtual int getHeapID() const;
    virtual void* getBase() const;
    virtual size_t getSize() const;
    virtual uint32_t getFlags() const;
    virtual uint32_t getOffset() const;
private:
    friend class IMemory;
    friend class HeapCache;
    // for debugging in this module
    static inline sp<IMemoryHeap> find_heap(const sp<IBinder>& binder) {
        return gHeapCache->find_heap(binder);
    }
    static inline void free_heap(const sp<IBinder>& binder) {
        gHeapCache->free_heap(binder);
    }
    static inline sp<IMemoryHeap> get_heap(const sp<IBinder>& binder) {
        return gHeapCache->get_heap(binder);
    }
    static inline void dump_heaps() {
        gHeapCache->dump_heaps();
    }
    void assertMapped() const;
    void assertReallyMapped() const;
    mutable std::atomic<int32_t> mHeapId;
    mutable void*       mBase;
    mutable size_t      mSize;
    mutable uint32_t    mFlags;
    mutable uint32_t    mOffset;
    mutable bool        mRealHeap;
    mutable Mutex       mLock;
};
```
先来看构造函数BpMemoryHeap的实现：
```cpp
BpMemoryHeap::BpMemoryHeap(const sp<IBinder>& impl)
    : BpInterface<IMemoryHeap>(impl),
        mHeapId(-1), mBase(MAP_FAILED), mSize(0), mFlags(0), mOffset(0), mRealHeap(false)
{
}
```
它的实现很简单，只是初始化一下各个成员变量，例如，表示匿名共享内存文件描述符的mHeapId值初化为-1、表示匿名内共享内存基地址的mBase值初始化为MAP_FAILED以及表示匿名共享内存大小的mSize初始为为0，它们都表示在Client端进程中，这个匿名共享内存还未准备就绪，要等到第一次使用时才会去创建。这里还需要注意的一点，参数impl指向的是一个BpBinder对象，它里面包含了一个指向Server端Binder对象，即MemoryHeapBase对象的引用。

以获取共享内存基地址为例，看看Client端如何获取服务端创建的共享内存。
```cpp
// frameworks/native/libs/binder/IMemory.cpp
void* BpMemoryHeap::getBase() const {
    assertMapped();
    return mBase;
}

void BpMemoryHeap::assertMapped() const
{
    int32_t heapId = mHeapId.load(memory_order_acquire);
    // 如果还没有请求服务创建匿名共享内存
    if (heapId == -1) {
        // 将当前BpMemoryHeap对象转换为IBinder对象
        sp<IBinder> binder(IInterface::asBinder(const_cast<BpMemoryHeap*>(this)));
        // 从成员变量gHeapCache中查找对应的BpMemoryHeap对象
        sp<BpMemoryHeap> heap(static_cast<BpMemoryHeap*>(find_heap(binder).get()));
        // 向服务端请求获取匿名共享内存信息
        heap->assertReallyMapped();
        // 判断该匿名共享内存是否映射成功
        if (heap->mBase != MAP_FAILED) {
            Mutex::Autolock _l(mLock);
            // 保存服务端返回回来的匿名共享内存信息
            if (mHeapId.load(memory_order_relaxed) == -1) {
                mBase   = heap->mBase;
                mSize   = heap->mSize;
                mOffset = heap->mOffset;
                int fd = fcntl(heap->mHeapId.load(memory_order_relaxed), F_DUPFD_CLOEXEC, 0);
                ALOGE_IF(fd==-1, "cannot dup fd=%d",
                        heap->mHeapId.load(memory_order_relaxed));
                mHeapId.store(fd, memory_order_release);
            }
        } else {
            // something went wrong
            free_heap(binder);
        }
    }
}
```
mHeapId等于-1，表示匿名共享内存还为准备就绪，因此请求服务端MemoryHeapBase创建匿名共享内存，否则该函数不作任何处理。只有第一次使用匿名共享时才会请求服务端创建匿名共享内存。由于在客户端进程中使用同一个BpBinder代理对象可以创建多个与匿名共享内存业务相关的BpMemoryHeap对象，因此定义了类型为HeapCache的全局变量gHeapCache用来保存创建的所有BpMemoryHeap对象，assertMapped函数首先将当前BpMemoryHeap对象强制转换为IBinder类型对象，然后调用find_heap()函数从全局变量gHeapCache中查找出对应的BpMemoryHeap对象，并调用assertReallyMapped()函数向服务进程的BnemoryHeap请求创建匿名共享内存。
```cpp
// frameworks/native/libs/binder/IMemory.cpp
void BpMemoryHeap::assertReallyMapped() const
{
    int32_t heapId = mHeapId.load(memory_order_acquire);
    // 再次判断是否已经请求创建过匿名共享内存
    if (heapId == -1) {
        // remote call without mLock held, worse case scenario, we end up
        // calling transact() from multiple threads, but that's not a problem,
        // only mmap below must be in the critical section.
        Parcel data, reply;
        data.writeInterfaceToken(IMemoryHeap::getInterfaceDescriptor());
        // 向服务端BnMemoryHeap发起请求
        status_t err = remote()->transact(HEAP_ID, data, &reply);
        int parcel_fd = reply.readFileDescriptor();
        ssize_t size = reply.readInt32();
        uint32_t flags = reply.readInt32();
        uint32_t offset = reply.readInt32();
        ALOGE_IF(err, "binder=%p transaction failed fd=%d, size=%zd, err=%d (%s)",
                IInterface::asBinder(this).get(),
                parcel_fd, size, err, strerror(-err));
        Mutex::Autolock _l(mLock);
        if (mHeapId.load(memory_order_relaxed) == -1) {
            int fd = fcntl(parcel_fd, F_DUPFD_CLOEXEC, 0);
            ALOGE_IF(fd==-1, "cannot dup fd=%d, size=%zd, err=%d (%s)",
                    parcel_fd, size, err, strerror(errno));
            int access = PROT_READ;
            if (!(flags & READ_ONLY)) {
                access |= PROT_WRITE;
            }
            mRealHeap = true;
            // 将服务进程创建的匿名共享内存映射到当前客户进程的地址空间中
            mBase = mmap(0, size, access, MAP_SHARED, fd, offset);
            if (mBase == MAP_FAILED) {
                ALOGE("cannot map BpMemoryHeap (binder=%p), size=%zd, fd=%d (%s)",
                        IInterface::asBinder(this).get(), size, fd, strerror(errno));
                close(fd);
            } else {
                // 映射成功后，将匿名共享内存信息保存到BpMemoryHeap的成员变量中，供其他接口函数访问
                mSize = size;
                mFlags = flags;
                mOffset = offset;
                mHeapId.store(fd, memory_order_release);
            }
        }
    }
}
```
该函数首先通过Binder通信方式向服务进程请求创建匿名共享内存，当服务端BnMemoryHeap对象创建完匿名共享内存后，并将共享内存信息返回到客户进程后，客户进程通过系统调用mmap函数将匿名共享内存映射到当前进程的地址空间，这样客户进程就可以访问服务进程创建的匿名共享内存了。当了解Binder通信机制，就知道BpMemoryHeap对象通过transact函数向服务端发起请求后，服务端的BnMemoryHeap的onTransact函数会被调用。

在上面获取共享内存的过程中，有一个类型为HeapCache的全局变量gHeapCache。
```cpp
// frameworks/native/libs/binder/IMemory.cpp
class HeapCache : public IBinder::DeathRecipient
{
public:
    HeapCache();
    virtual ~HeapCache();
    virtual void binderDied(const wp<IBinder>& who);
    sp<IMemoryHeap> find_heap(const sp<IBinder>& binder);
    void free_heap(const sp<IBinder>& binder);
    sp<IMemoryHeap> get_heap(const sp<IBinder>& binder);
    void dump_heaps();
private:
    // For IMemory.cpp
    struct heap_info_t {
        sp<IMemoryHeap> heap;
        int32_t         count;
        // Note that this cannot be meaningfully copied.
    };
    void free_heap(const wp<IBinder>& binder);
    Mutex mHeapCacheLock;  // Protects entire vector below.
    KeyedVector< wp<IBinder>, heap_info_t > mHeapCache;
    // We do not use the copy-on-write capabilities of KeyedVector.
    // TODO: Reimplemement based on standard C++ container?
};
static sp<HeapCache> gHeapCache = new HeapCache();
```
它里面定义了一个成员变量mHeapCache，用来维护本进程中的所有BpMemoryHeap对象，同时还提供了find_heap和get_heap函数来查找内部所维护的BpMemoryHeap对象的功能。函数find_heap和get_heap的区别是，在find_heap函数中，如果在mHeapCache找不到相应的BpMemoryHeap对象，就会把这个BpMemoryHeap对象加入到mHeapCache中去，而在get_heap函数中，则不会自动把这个BpMemoryHeap对象加入到mHeapCache中去。

这里，我们主要看一下find_heap函数的实现：
```cpp
// frameworks/native/libs/binder/IMemory.cpp
sp<IMemoryHeap> HeapCache::find_heap(const sp<IBinder>& binder)
{
    Mutex::Autolock _l(mHeapCacheLock);
    ssize_t i = mHeapCache.indexOfKey(binder);
    if (i>=0) {
        heap_info_t& info = mHeapCache.editValueAt(i);
        ALOGD_IF(VERBOSE,
                "found binder=%p, heap=%p, size=%zu, fd=%d, count=%d",
                binder.get(), info.heap.get(),
                static_cast<BpMemoryHeap*>(info.heap.get())->mSize,
                static_cast<BpMemoryHeap*>(info.heap.get())
                    ->mHeapId.load(memory_order_relaxed),
                info.count);
        ++info.count;
        return info.heap;
    } else {
        heap_info_t info;
        info.heap = interface_cast<IMemoryHeap>(binder);
        info.count = 1;
        //ALOGD("adding binder=%p, heap=%p, count=%d",
        //      binder.get(), info.heap.get(), info.count);
        mHeapCache.add(binder, info);
        return info.heap;
    }
}
```
 这个函数很简单，首先它以传进来的参数binder为关键字，在mHeapCache中查找，看看是否有对应的heap_info对象info存在，如果有的话，就增加它的引用计数info.count值，表示这个BpBinder对象多了一个使用者；如果没有的话，那么就需要创建一个heap_info对象info，并且将它加放到mHeapCache中去了。
### MemoryBase
MemoryBase接口是建立在MemoryHeapBase接口的基础上的，它们都可以作为一个Binder对象来在进程间进行数据共享，它们的关系如下所示：
![](https://i.imgur.com/Ws4jiXt.png)
MemoryBase类包含了一个成员变量mHeap，它的类型的IMemoryHeap，MemoryBase类所代表的匿名共享内存就是通过这个成员变量来实现的。

MemoryBase类在Server端的实现与MemoryHeapBase类在Server端的实现是类似的，这里只要把IMemory类换成IMemoryHeap类、把BnMemory类换成BnMemoryHeap类以及MemoryBase类换成MemoryHeapBase类就变成是MemoryHeapBase类在Server端的实现了，因此，我们这里只简单分析IMemory类和MemoryBase类的实现。
```cpp
class IMemory : public IInterface
{
public:
    DECLARE_META_INTERFACE(Memory)
    virtual sp<IMemoryHeap> getMemory(ssize_t* offset=0, size_t* size=0) const = 0;
    // helpers
    void* fastPointer(const sp<IBinder>& heap, ssize_t offset) const;
    void* pointer() const;
    size_t size() const;
    ssize_t offset() const;
};
```
成员函数getMemory用来获取内部的MemoryHeapBase对象的IMemoryHeap接口；成员函数pointer()用来获取内部所维护的匿名共享内存的基地址；成员函数size()用来获取内部所维护的匿名共享内存的大小；成员函数offset()用来获取内部所维护的这部分匿名共享内存在整个匿名共享内存中的偏移量。

IMemory类本身实现了pointer、size和offset三个成员函数，因此，它的子类，即MemoryBase类，只需要实现getMemory成员函数就可以了。IMemory类的实现定义在frameworks/native/libs/binder/IMemory.cpp文件中：
```cpp
void* IMemory::fastPointer(const sp<IBinder>& binder, ssize_t offset) const
{
    sp<IMemoryHeap> realHeap = BpMemoryHeap::get_heap(binder);
    void* const base = realHeap->base();
    if (base == MAP_FAILED)
        return 0;
    return static_cast<char*>(base) + offset;
}
void* IMemory::pointer() const {
    ssize_t offset;
    sp<IMemoryHeap> heap = getMemory(&offset);
    void* const base = heap!=0 ? heap->base() : MAP_FAILED;
    if (base == MAP_FAILED)
        return 0;
    return static_cast<char*>(base) + offset;
}
size_t IMemory::size() const {
    size_t size;
    getMemory(NULL, &size);
    return size;
}
ssize_t IMemory::offset() const {
    ssize_t offset;
    getMemory(&offset);
    return offset;
}
```
当客户端的BpMemory向服务端MemoryBase发起RPC请求后，服务端的BnMemory对象的onTransact函数被调用
```cpp
status_t BnMemory::onTransact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    switch(code) {
        case GET_MEMORY: {
            // 根据客户端发送过来的接口描述进行检查确认
            CHECK_INTERFACE(IMemory, data, reply);
            ssize_t offset;
            size_t size;
            // 调用服务端的getMemory函数获取匿名共享内存对象MemoryHeapBase
            // 及匿名共享内存大小，偏移，并返回给客户端
            reply->writeStrongBinder( IInterface::asBinder(getMemory(&offset, &size)) );
            // 将偏移量返回给客户端
            reply->writeInt32(offset);
             // 将匿名共享内存大小返回给客户端
            reply->writeInt32(size);
            return NO_ERROR;
        } break;
        default:
            return BBinder::onTransact(code, data, reply, flags);
    }
}
```
服务端的getMemory函数由BnMemory的子类MemoryBase实现。
```cpp
sp<IMemoryHeap> MemoryBase::getMemory(ssize_t* offset, size_t* size) const
{
    if (offset) *offset = mOffset;
    if (size)   *size = mSize;
    return mHeap;
}

MemoryBase::MemoryBase(const sp<IMemoryHeap>& heap,
        ssize_t offset, size_t size)
    : mSize(size), mOffset(offset), mHeap(heap)
{
}
```
在它的构造函数中，接受三个参数，参数heap指向的是一个MemoryHeapBase对象，真正的匿名共享内存就是由它来维护的，参数offset表示这个MemoryBase对象所要维护的这部分匿名共享内存在整个匿名共享内存块中的起始位置，参数size表示这个MemoryBase对象所要维护的这部分匿名共享内存的大小。

成员函数getMemory的实现很简单，只是简单地返回内部的MemoryHeapBase对象的IMemoryHeap接口，如果传进来的参数offset和size不为NULL，还会把其内部维护的这部分匿名共享内存在整个匿名共享内存块中的偏移位置以及这部分匿名共享内存的大小返回给调用者。

这里可以看出，MemoryBase在Server端的实现只是简单地封装了MemoryHeapBase的实现。

MemoryBase类在Client端的实现与MemoryHeapBase类在Client端的实现是类似的，这里只要把IMemory类换成IMemoryHeap类以及把BpMemory类换成BpMemoryHeap类就变成是MemoryHeapBase类在Client端的实现了，因此，我们这里只简单分析BpMemory类的实现，前面已经分析过IMemory类的实现了。
```cpp
class BpMemory : public BpInterface<IMemory>
{
public:
    explicit BpMemory(const sp<IBinder>& impl);
    virtual ~BpMemory();
    virtual sp<IMemoryHeap> getMemory(ssize_t* offset=0, size_t* size=0) const;
private:
    mutable sp<IMemoryHeap> mHeap;
    mutable ssize_t mOffset;
    mutable size_t mSize;
};
```
下面就看一下BpMemory类的成员函数getMemory的实现：
```cpp
sp<IMemoryHeap> BpMemory::getMemory(ssize_t* offset, size_t* size) const
{
    // 指向的匿名共享内存MemoryHeapBase为空
    if (mHeap == 0) {
        Parcel data, reply;
        data.writeInterfaceToken(IMemory::getInterfaceDescriptor());
        // 向服务端MemoryBase发起RPC请求
        if (remote()->transact(GET_MEMORY, data, &reply) == NO_ERROR) {
            // 读取服务端返回来的结果
            // 读取匿名共享内存MemoryHeapBase的IBinder对象
            sp<IBinder> heap = reply.readStrongBinder(); 
            ssize_t o = reply.readInt32(); // 读取匿名共享内存中的偏移量
            size_t s = reply.readInt32();  // 读取匿名共享内存的大小
            // 如果服务端返回来的用于描述整块匿名共享内存的MemoryHeapBase不为空
            if (heap != 0) {
                mHeap = interface_cast<IMemoryHeap>(heap);
                if (mHeap != 0) {
                    size_t heapSize = mHeap->getSize();
                    // 将匿名共享内存的偏移和大小保存到成员变量中
                    if (s <= heapSize
                            && o >= 0
                            && (static_cast<size_t>(o) <= heapSize - s)) {
                        mOffset = o;
                        mSize = s;
                    } else {
                        // Hm.
                        android_errorWriteWithInfoLog(0x534e4554,
                            "26877992", -1, NULL, 0);
                        mOffset = 0;
                        mSize = 0;
                    }
                }
            }
        }
    }
    // 将成员变量赋值给传进来的参数，从而修改参数值
    if (offset) *offset = mOffset;
    if (size) *size = mSize;
    return (mSize > 0) ? mHeap : 0;
}
```
如果成员变量mHeap的值为NULL，就表示这个BpMemory对象尚未建立好匿名共享内存，于是，就会通过一个Binder进程间调用去Server端请求匿名共享内存信息，在这些信息中，最重要的就是这个Server端的MemoryHeapBase对象的引用heap了，通过这个引用可以在Client端进程中创建一个BpMemoryHeap远程接口，最后将这个BpMemoryHeap远程接口保存在成员变量mHeap中，同时，从Server端获得的信息还包括这块匿名共享内存在整个匿名共享内存中的偏移位置以及大小。这样，这个BpMemory对象中的匿名共享内存就准备就绪了。
### MemoryDealer
在AudioFlinger创建Client过程中，使用MemoryDealer申请共享内存,其实MemoryDealer工具类就是对MemoryHeapBase和MemoryBase的封装。
```cpp
mMemoryDealer = new MemoryDealer(heapSize, "AudioFlinger::Client");
```
下面看看MemoryDealer的构造函数，这里会创建MemoryHeapBase，即分配4M共享内存，还会创建与一个简单分配器，后续每次通信需要的具体内存由SimpleBestFitAllocator从4M内存上分配。在SimpleBestFitAllocator中维护一个双向链表来管理这4M的内存。
```cpp
MemoryDealer::MemoryDealer(size_t size, const char* name, uint32_t flags)
    : mHeap(new MemoryHeapBase(size, flags, name)),
    mAllocator(new SimpleBestFitAllocator(size))
{    
}

// align all the memory blocks on a cache-line boundary
const int SimpleBestFitAllocator::kMemoryAlign = 32;
SimpleBestFitAllocator::SimpleBestFitAllocator(size_t size)
{
    size_t pagesize = getpagesize();
    mHeapSize = ((size + pagesize-1) & ~(pagesize-1));
    chunk_t* node = new chunk_t(0, mHeapSize / kMemoryAlign);
    mList.insertHead(node);
}
```
前面我们看到最终申请内存通过MemoryDealer的allocate的函数，返回的是IMemory类型。从代码看，会返回Allocation，继承自MemoryBase。分配时，最终调用SimpleBestFitAllocator的alloc函数。
```cpp
sp<IMemory> MemoryDealer::allocate(size_t size)
{
    sp<IMemory> memory;
    const ssize_t offset = allocator()->allocate(size);
    if (offset >= 0) {
        memory = new Allocation(this, heap(), offset, size);
    }
    return memory;
}

SimpleBestFitAllocator* MemoryDealer::allocator() const {
    return mAllocator;
}

size_t SimpleBestFitAllocator::allocate(size_t size, uint32_t flags)
{
    Mutex::Autolock _l(mLock);
    ssize_t offset = alloc(size, flags);
    return offset;
}

ssize_t SimpleBestFitAllocator::alloc(size_t size, uint32_t flags)
{
    if (size == 0) {
        return 0;
    }
    size = (size + kMemoryAlign-1) / kMemoryAlign;
    chunk_t* free_chunk = 0;
    chunk_t* cur = mList.head();
    size_t pagesize = getpagesize();
    while (cur) {
        int extra = 0;
        if (flags & PAGE_ALIGNED)
            extra = ( -cur->start & ((pagesize/kMemoryAlign)-1) ) ;
        // best fit
        if (cur->free && (cur->size >= (size+extra))) {
            if ((!free_chunk) || (cur->size < free_chunk->size)) {
                free_chunk = cur;
            }
            if (cur->size == size) {
                break;
            }
        }
        cur = cur->next;
    }
    if (free_chunk) {
        const size_t free_size = free_chunk->size;
        free_chunk->free = 0;
        free_chunk->size = size;
        if (free_size > size) {
            int extra = 0;
            if (flags & PAGE_ALIGNED)
                extra = ( -free_chunk->start & ((pagesize/kMemoryAlign)-1) ) ;
            if (extra) {
                chunk_t* split = new chunk_t(free_chunk->start, extra);
                free_chunk->start += extra;
                mList.insertBefore(free_chunk, split);
            }
            ALOGE_IF((flags&PAGE_ALIGNED) && 
                    ((free_chunk->start*kMemoryAlign)&(pagesize-1)),
                    "PAGE_ALIGNED requested, but page is not aligned!!!");
            const ssize_t tail_free = free_size - (size+extra);
            if (tail_free > 0) {
                chunk_t* split = new chunk_t(
                        free_chunk->start + free_chunk->size, tail_free);
                mList.insertAfter(free_chunk, split);
            }
        }
        return (free_chunk->start)*kMemoryAlign;
    }
    return NO_MEMORY;
}
```
链表节点，每次从4M空间挖取一段内存作为一个节点，不用时再释放删除节点，通过这种简单的方式来管理这段内存。
```cpp
struct chunk_t {
        chunk_t(size_t start, size_t size)
        : start(start), size(size), free(1), prev(0), next(0) {
        }
        size_t              start;                  // 起始地址
        size_t              size : 28;              // 内存大小
        int                 free : 4;               // 是否占用
        mutable chunk_t*    prev;
        mutable chunk_t*    next;
    };
```
## AudioTrack数据写入和AudioFlinger数据读取
AudioTrack实例构造后，应用程序接着可以写入音频数据了。AudioTrack与AudioFlinger是生产者-消费者的关系：
- AudioTrack：AudioTrack在共享内存中找到一块可用空间，把用户传入的音频数据写入到这块可用空间上，然后更新写位置（对于AudioFinger来说，意味共享内存上有更多的可读数据了）；如果用户传入的数据量比可用空间要大，那么要把用户传入的数据拆分多次写入到共享中（AudioTrack和AudioFlinger是不同的进程，AudioFlinger同时也在不停地读取数据，所以共享内存可用空间是在不停变化的）。
- AudioFlinger：AudioFlinger在共享内存中找到一块可读数据块，把可读数据拷贝到目的缓冲区上，然后更新读位置（对于AudioTrack来说，意味着共享内存上有更多的可用空间了）；如果共享内存上可读数据量比预期的要小，那么要进行多次的读取，才能积累到预期的数据量（AudioTrack和AudioFlinger是不同的进程，AudioTrack同时也在不停地写入数据，所以共享内存可读的数据量是在不停变化的）。

在AudioTrack和AudioFlinger操作共享内存的时使用Proxy来管理。
```cpp
// frameworks/av/include/private/meida/AudioTrackShared.h
// Proxy for shared memory control block, to isolate callers 
// from needing to know the details. There is exactly one
// ClientProxy and one ServerProxy per shared memory control block.
// The proxies are located in normal memory, 
// and are not multi-thread safe within a given side.
class Proxy : public RefBase {
protected:
    Proxy(audio_track_cblk_t* cblk, void *buffers, size_t frameCount, 
          size_t frameSize, bool isOut, bool clientInServer);
    virtual ~Proxy() { }

public:
    size_t frameCount() const { return mFrameCount; }

protected:
    // These refer to shared memory, and are virtual addresses with respect to the 
    // current process. They may have different virtual addresses within the other process.
    audio_track_cblk_t* const   mCblk;  // the control block
    void* const     mBuffers;           // starting address of buffers

    const size_t    mFrameCount;        // not necessarily a power of 2
    const size_t    mFrameSize;         // in bytes
    const size_t    mFrameCountP2;      // mFrameCount rounded to power of 2, streaming mode
    const bool      mIsOut;             // true for AudioTrack, false for AudioRecord
    const bool      mClientInServer;    // true for OutputTrack, 
                                        // false for AudioTrack & AudioRecord
    bool            mIsShutdown;        // latch set to true when 
                                        // shared memory corruption detected
    size_t          mUnreleased;        // unreleased frames remaining 
                                        // from most recent obtainBuffer
};
```
如下是Proxy的类图：
![](https://i.imgur.com/z0xTk2d.png)
- AudioTrackClientProxy：MODE_STREAM模式下，生产者AudioTrack使用它在共享内存中找到可用空间的位置
- AudioTrackServerProxy：MODE_STREAM模式下，消费者AudioFlinger::PlaybackThread使用它在 共享内存中找到可读数据的位置

- StaticAudioTrackClientProxy：MODE_STATIC模式下，生产者AudioTrack使用它在共享内存中找到可用空间的位置
- StaticAudioTrackServerProxy：MODE_STATIC模式下，消费者AudioFlinger::PlaybackThread 使用它在共享内存中找到可读数据的位置

- AudioRecordClientProxy：消费者AudioRecord使用它在共享内存中找到可读数据的位置
- AudioTrackServerProxy：生产者AudioFlinger::RecordThread使用它在共享内存中找到可用空间的位置

### AudioTrack数据写入
在写数据的过程中会用到两个buffer，分别是AudioTrack::Buffer和Proxy::Buffer，它们声明如下。
```cpp
// frameworks/av/media/libaudioclient/include/media/AudioTrack.h
// AudioTrack::Buffer声明
class Buffer
{
public:
    // FIXME use m prefix
    size_t      frameCount;   // number of sample frames corresponding to size;
                              // on input to obtainBuffer() it is the number of frames desired,
                              // on output from obtainBuffer() it is the number of available
                              //    [empty slots for] frames to be filled
                              // on input to releaseBuffer() it is currently ignored
    size_t      size;         // input/output in bytes == frameCount * frameSize
                              // on input to obtainBuffer() it is ignored
                              // on output from obtainBuffer() it is the number of available
                              //    [empty slots for] bytes to be filled,
                              //    which is frameCount * frameSize
                              // on input to releaseBuffer() it is the number of bytes to
                              //    release
                              // FIXME This is redundant with respect to frameCount.  Consider
                              //    removing size and making frameCount the primary field.
    union {
        void*       raw;
        short*      i16;      // signed 16-bit
        int8_t*     i8;       // unsigned 8-bit, offset by 0x80
    };                        // input to obtainBuffer(): unused, output: pointer to buffer
};

// frameworks/av/include/private/meida/AudioTrackShared.h
// Proxy::Buffer声明
struct Buffer {
    size_t  mFrameCount;            // number of frames available in this buffer
    void*   mRaw;                   // pointer to first frame
    size_t  mNonContig;             // number of additional non-contiguous frames available
};
```
当创建好AudioTrack，Client端会执行写数据，则会调用AudioTrack的write函数向硬件写数据，首先会获取可用的共享内存空间，将数据拷贝到这块可用空间，然后更新共享内存写数据的位置。
```cpp
// frameworks/av/media/libaudioclient/AudioTrack.cpp
ssize_t AudioTrack::write(const void* buffer, size_t userSize, bool blocking)
{
    // 条件检查
    ......
    size_t written = 0;
    Buffer audioBuffer; //如上声明
    while (userSize >= mFrameSize) {
        // 单帧数据量 frameSize = channelCount * bytesPerSample
        // 对于双声道，16位采样的音频数据来说，frameSize = 2 * 2 = 4(bytes)
        // 用户传入的数据帧数 frameCount = userSize / frameSize
        audioBuffer.frameCount = userSize / mFrameSize;
        // obtainBuffer() 从共享内存上得到一块可用区间
        status_t err = obtainBuffer(&audioBuffer,
                blocking ? &ClientProxy::kForever : &ClientProxy::kNonBlocking);
        
        ......
        // toWrite 是共享内存上可用区间的大小，可能比userSize（用户传入数据的大小）要小
        // 因此用户传入的数据可能要拆分多次拷贝到共享内存上
        // 注意：AudioTrack和AudioFlinger是不同的进程，AudioFlinger同时也在不停地
        // 消耗数据，所以共享内存可用区间是在不停变化的
        size_t toWrite = audioBuffer.size;
        memcpy(audioBuffer.i8, buffer, toWrite);    // 把用户数据拷贝到共享内存可用区间
        buffer = ((const char *) buffer) + toWrite; // 未拷贝数据的位置
        userSize -= toWrite;                        // 未拷贝数据的大小
        written += toWrite;                         // 已拷贝数据的大小
        
        // releaseBuffer() 更新共享内存写位置
        // 对于AudioFinger来说，意味共享内存上有更多的可读数据
        releaseBuffer(&audioBuffer);
    }
    if (written > 0) {
        mFramesWritten += written / mFrameSize;
    }
    return written;
}
```
在AudioTrack内部会先设置进程睡眠时间，然后调用AudioTrackClientProxy的obtainBuffer函数获取Proxy::Buffer，然后将其转换为AudioTrack::Buffer，然后通过参数返回。
```cpp
// frameworks/av/media/libaudioclient/AudioTrack.cpp
status_t AudioTrack::obtainBuffer(Buffer* audioBuffer, int32_t waitCount, size_t *nonContig)
{
    // 条件检查
    .....
    const struct timespec *requested;
    struct timespec timeout;
    // 通过waitCount的值计算是否需要等待，若要等待则等待多久
    .....
    return obtainBuffer(audioBuffer, requested, NULL /*elapsed*/, nonContig);
}

status_t AudioTrack::obtainBuffer(Buffer* audioBuffer, const struct timespec *requested,
        struct timespec *elapsed, size_t *nonContig)
{
    // previous and new IAudioTrack sequence numbers are used to detect track re-creation
    uint32_t oldSequence = 0;
    uint32_t newSequence;
    Proxy::Buffer buffer;
    status_t status = NO_ERROR;
    static const int32_t kMaxTries = 5; // 最多5次尝试获取可用空间
    int32_t tryCounter = kMaxTries;
    do {
        // obtainBuffer() is called with mutex unlocked, so keep extra references 
        // to these fields to keep them from going away 
        // if another thread re-creates the track during obtainBuffer()
        sp<AudioTrackClientProxy> proxy;
        sp<IMemory> iMem;

        ...... // 对异常状况，stop状态等情况的处理，并获取AudioTrackClientProxy

        // 调用AudioTrackClientProxy的obtainBuffer函数，由于继承关系，
		// 会调用ClientProxy的obtainBuffer函数
        buffer.mFrameCount = audioBuffer->frameCount;
        // FIXME starts the requested timeout and elapsed over from scratch
        status = proxy->obtainBuffer(&buffer, requested, elapsed);
    } while (((status == DEAD_OBJECT) || (status == NOT_ENOUGH_DATA)) && (tryCounter-- > 0));

    将Proxy::Buffer转为AudioTrack::Buffer
    audioBuffer->frameCount = buffer.mFrameCount;
    audioBuffer->size = buffer.mFrameCount * mFrameSize;
    audioBuffer->raw = buffer.mRaw;
    if (nonContig != NULL) {
        *nonContig = buffer.mNonContig;
    }
    return status;
}

```
在ClientProxy中，首先设置timeout类型，然后从共享内存读取队头和队尾指针，从而计算已经使用的区域，结合共享内存的大小计算可用空间，最后找到一块合适的空间返回。在其中还会通过系统调用进行进程间的同步控制，看具体情况将进程挂起。
```cpp
// frameworks/av/media/libaudioclient/AudioTrackShared.cpp
__attribute__((no_sanitize("integer")))
status_t ClientProxy::obtainBuffer(Buffer* buffer, const struct timespec *requested,
        struct timespec *elapsed)
{
    LOG_ALWAYS_FATAL_IF(buffer == NULL || buffer->mFrameCount == 0);
    struct timespec total;          // total elapsed time spent waiting
    total.tv_sec = 0;
    total.tv_nsec = 0;
    bool measure = elapsed != NULL; // whether to measure total elapsed time spent waiting
    status_t status;
    enum {
        TIMEOUT_ZERO,       // requested == NULL || *requested == 0
        TIMEOUT_INFINITE,   // *requested == infinity
        TIMEOUT_FINITE,     // 0 < *requested < infinity
        TIMEOUT_CONTINUE,   // additional chances after TIMEOUT_FINITE
    } timeout;

    ...... // 根据requested设置timeout类型

    struct timespec before;
    bool beforeIsValid = false;
    audio_track_cblk_t* cblk = mCblk;
    bool ignoreInitialPendingInterrupt = true;
    // check for shared memory corruption
    if (mIsShutdown) {
        status = NO_INIT;
        goto end;
    }
    for (;;) {

        ...... // 基本条件检查

        int32_t front;
        int32_t rear;
        // 注意使用带内存屏障的函数android_atomic_acquire_load
        // 获取队头和队尾
        if (mIsOut) { // 对应AudioTrack
            // The barrier following the read of mFront is probably redundant.
            // We're about to perform a conditional branch based on 'filled',
            // which will force the processor to observe the read of mFront
            // prior to allowing data writes starting at mRaw.
            // However, the processor may support speculative execution,
            // and be unable to undo speculative writes into shared memory.
            // The barrier will prevent such speculative execution.
            front = android_atomic_acquire_load(&cblk->u.mStreaming.mFront);
            rear = cblk->u.mStreaming.mRear;
        } else { // 对应AudioRecord
            // On the other hand, this barrier is required.
            rear = android_atomic_acquire_load(&cblk->u.mStreaming.mRear);
            front = cblk->u.mStreaming.mFront;
        }
        // write to rear, read from front
        // 已经使用的区域
        ssize_t filled = rear - front;
        // pipe should not be overfull
        // 当已经使用的空间大于预先设置的帧数，对于播放来讲出错了，
        // 而对于录音来说目前处于overrun,写入的速度太快，读取熟读跟不上
        if (!(0 <= filled && (size_t) filled <= mFrameCount)) {
            if (mIsOut) {
                ALOGE("Shared memory control block is corrupt (filled=%zd, mFrameCount=%zu); "
                        "shutting down", filled, mFrameCount);
                mIsShutdown = true;
                status = NO_INIT;
                goto end;
            }
            // for input, sync up on overrun
            filled = 0;
            cblk->u.mStreaming.mFront = rear;
            (void) android_atomic_or(CBLK_OVERRUN, &cblk->mFlags);
        }
        // Don't allow filling pipe beyond the user settable size.
        // The calculation for avail can go negative if the buffer size
        // is suddenly dropped below the amount already in the buffer.
        // So use a signed calculation to prevent a numeric overflow abort.
        // 有符号运算可用空间
        ssize_t adjustableSize = (ssize_t) getBufferSizeInFrames();
        ssize_t avail =  (mIsOut) ? adjustableSize - filled : filled;
        if (avail < 0) {
            avail = 0;
        } else if (avail > 0) {
            // 'avail' may be non-contiguous, so return only the first contiguous chunk
            size_t part1;
            if (mIsOut) {
                rear &= mFrameCountP2 - 1;
                part1 = mFrameCountP2 - rear;
            } else {
                front &= mFrameCountP2 - 1;
                part1 = mFrameCountP2 - front;
            }
            if (part1 > (size_t)avail) {
                part1 = avail;
            }
            if (part1 > buffer->mFrameCount) {
                part1 = buffer->mFrameCount;
            }

            // 赋值Proxy::Buffer
            buffer->mFrameCount = part1;
            buffer->mRaw = part1 > 0 ?
                    &((char *) mBuffers)[(mIsOut ? rear : front) * mFrameSize] : NULL;
            buffer->mNonContig = avail - part1;
            mUnreleased = part1;
            status = NO_ERROR;
            break;
        }
        struct timespec remaining;
        const struct timespec *ts;

        ...... // 根据timeout类型计算剩余等待的时间ts

        int32_t old = android_atomic_and(~CBLK_FUTEX_WAKE, &cblk->mFutex);
        if (!(old & CBLK_FUTEX_WAKE)) {
            if (measure && !beforeIsValid) {
                clock_gettime(CLOCK_MONOTONIC, &before);
                beforeIsValid = true;
            }
            errno = 0;
            // 同步控制，down (P)操作,原子性的给cblk->mFutex同步变量减1，
            // FUTEX_WAIT，原子性的检查cblk->mFutex中计数器的值是否为负值,
            // 如果是则让进程休眠，直到FUTEX_WAKE或者超时(time-out)。也就是
            // 把进程挂到cblk->mFutex相对应的等待队列上去。
            (void) syscall(__NR_futex, &cblk->mFutex,
                    mClientInServer ? FUTEX_WAIT_PRIVATE : FUTEX_WAIT, 
                    old & ~CBLK_FUTEX_WAKE, ts);
            status_t error = errno; // clock_gettime can affect errno
            // update total elapsed time spent waiting
            ......
            switch (error) {
            case 0:            // normal wakeup by server, or by binderDied()
            case EWOULDBLOCK:  // benign race condition with server
            case EINTR:        // wait was interrupted by signal or other spurious wakeup
            case ETIMEDOUT:    // time-out expired
                // FIXME these error/non-0 status are being dropped
                break;
            default:
                status = error;
                ALOGE("%s unexpected error %s", __func__, strerror(status));
                goto end;
            }
        }
    }
end:
    ...... // 善后处理
    return status;
}
```
Client端拷贝完会更新共享内存的写数据指针，这一步通过ClientProxy的releaseBuffer实现。
```cpp
// frameworks/av/media/libaudioclient/AudioTrack.cpp
void AudioTrack::releaseBuffer(const Buffer* audioBuffer)
{
    // FIXME add error checking on mode, by adding an internal version
    ..... // 条件检查

    // 数据写入完毕，将数据从AudioTrack::Buffer转入Proxy::Buffer，
    // 调用AudioTrackClientProxy的releaseBuffer函数释放buffer控制权
    Proxy::Buffer buffer;
    buffer.mFrameCount = stepCount;
    buffer.mRaw = audioBuffer->raw;
    AutoMutex lock(mLock);
    mReleased += stepCount;
    mInUnderrun = false;
    mProxy->releaseBuffer(&buffer);
    // restart track if it was disabled by audioflinger due to previous underrun
    restartIfDisabled();
}

// frameworks/av/media/libaudioclient/AudioTrackShared.cpp
__attribute__((no_sanitize("integer")))
void ClientProxy::releaseBuffer(Buffer* buffer)
{
    LOG_ALWAYS_FATAL_IF(buffer == NULL);
    size_t stepCount = buffer->mFrameCount;
    if (stepCount == 0 || mIsShutdown) {
        // prevent accidental re-use of buffer
        buffer->mFrameCount = 0;
        buffer->mRaw = NULL;
        buffer->mNonContig = 0;
        return;
    }
    LOG_ALWAYS_FATAL_IF(!(stepCount <= mUnreleased && mUnreleased <= mFrameCount));
    mUnreleased -= stepCount;
    audio_track_cblk_t* cblk = mCblk;
    // Both of these barriers are required
    // 释放buffer控制权，其实是移动队尾（队头），更新写数据位置，注意内存屏障的使用
    if (mIsOut) {
        int32_t rear = cblk->u.mStreaming.mRear;
        android_atomic_release_store(stepCount + rear, &cblk->u.mStreaming.mRear);
    } else {
        int32_t front = cblk->u.mStreaming.mFront;
        android_atomic_release_store(stepCount + front, &cblk->u.mStreaming.mFront);
    }
}
```
### AudioFlinger数据读取
在AudioFlinger端读取数据会使用到AudioBufferProvider::Buffer和Proxy::Buffer。
```cpp
// frameworks/av/media/libaudioclient/include/media/AudioBufferProvider.h
AudioBufferProvider::Buffer声明
struct Buffer {
    Buffer() : raw(NULL), frameCount(0) { }
    union {
        void*       raw;
        short*      i16;
        int8_t*     i8;
    };
    size_t frameCount;
};

// frameworks/av/include/private/meida/AudioTrackShared.h
// Proxy::Buffer声明
struct Buffer {
    size_t  mFrameCount;            // number of frames available in this buffer
    void*   mRaw;                   // pointer to first frame
    size_t  mNonContig;             // number of additional non-contiguous frames available
};
```
我们以DirectOutputThread/OffloadThread为例说明（MixerThread读数据也是类似的过程，由于MixerThread会有混音过程，所以读取数据会稍微复杂点，是在AudioMixer中进行的，后续有机会分析混音时在分析其读取数据的过程）。
```cpp
// frameworks/av/services/audioflinger/Threads.cpp
void AudioFlinger::DirectOutputThread::threadLoop_mix()
{
    // mFrameCount是硬件设备（PCM设备）处理单个数据块的帧数（周期大小）
    // 上层必须积累了足够多（mFrameCount）的数据，才写入到PCM设备所以
    // mFrameCount也就是AudioFlinger预期的数据量
    size_t frameCount = mFrameCount;
    // mSinkBuffer目的缓冲区，threadLoop_write() 会把mSinkBuffer上的数据写到PCM设备
    int8_t *curBuf = (int8_t *)mSinkBuffer;
    // output audio to hardware
    // FIFO 上可读的数据量可能要比预期的要小，因此可能需要多次读取才能积累足够的数据量
    // 注意：AudioTrack和AudioFlinger是不同的进程，AudioTrack同时也在不停地生产数据
    // 所以共享内存可读的数据量是在不停变化的
    while (frameCount) {
        AudioBufferProvider::Buffer buffer;
        buffer.frameCount = frameCount;
        // getNextBuffer() 从共享内存上获取可读数据块
        status_t status = mActiveTrack->getNextBuffer(&buffer);
        if (status != NO_ERROR || buffer.raw == NULL) {
            // no need to pad with 0 for compressed audio
            if (audio_has_proportional_frames(mFormat)) {
                memset(curBuf, 0, frameCount * mFrameSize);
            }
            break;
        }
        // memcpy()把共享内存可读数据拷贝到mSinkBuffer目的缓冲区
        memcpy(curBuf, buffer.raw, buffer.frameCount * mFrameSize);
        frameCount -= buffer.frameCount;
        curBuf += buffer.frameCount * mFrameSize;
        // releaseBuffer()更新共享内存读位置
        // 对于AudioTrack来说，意味着共享内存上有更多的可用空间
        mActiveTrack->releaseBuffer(&buffer);
    }
    mCurrentWriteLength = curBuf - (int8_t *)mSinkBuffer;
    mSleepTimeUs = 0;
    mStandbyTimeNs = systemTime() + mStandbyDelayNs;
    mActiveTrack.clear();
}
```
从上面看到会先通过mActiveTrack的getNextBuffer获取可读数据，即Track类的getNextBuffer函数，然后将数据拷贝到目的mSinkBuffer，然后更新读数据指针位置。
```cpp
// frameworks/av/services/audioflinger/Tracks.cpp
// AudioBufferProvider interface
status_t AudioFlinger::PlaybackThread::Track::getNextBuffer(
        AudioBufferProvider::Buffer* buffer)
{
    ServerProxy::Buffer buf;
    size_t desiredFrames = buffer->frameCount;
    buf.mFrameCount = desiredFrames;
    // 调用AudioTrackServerProxy的obtainBuffer函数，由于继承关系，
	// 会调用ServerProxy的obtainBuffer函数 
    status_t status = mServerProxy->obtainBuffer(&buf);
    // Proxy::Buffer转化为AudioBufferProvider::Buffer
    buffer->frameCount = buf.mFrameCount;
    buffer->raw = buf.mRaw;
    // 是否处于underrun状态
    if (buf.mFrameCount == 0 && !isStopping() && !isStopped() && !isPaused()) {
        ALOGV("underrun,  framesReady(%zu) < framesDesired(%zd), state: %d",
                buf.mFrameCount, desiredFrames, mState);
        mAudioTrackServerProxy->tallyUnderrunFrames(desiredFrames);
    } else {
        mAudioTrackServerProxy->tallyUnderrunFrames(0);
    }
    return status;
}

// AudioBufferProvider interface
// getNextBuffer() = 0;
// This implementation of releaseBuffer() is used by Track and RecordTrack
void AudioFlinger::ThreadBase::TrackBase::releaseBuffer(AudioBufferProvider::Buffer* buffer)
{
    // AudioBufferProvider::Buffer转化为Proxy::Buffer
    ServerProxy::Buffer buf;
    buf.mFrameCount = buffer->frameCount;
    buf.mRaw = buffer->raw;
    buffer->frameCount = 0;
    buffer->raw = NULL;
    // 调用AudioTrackServerProxy的releaseBuffer函数，由于继承关系，
	// 会调用ServerProxy的releaseBuffer函数 
    mServerProxy->releaseBuffer(&buf);
}
```
获取可读数据及更新读数据指针位置最终会通过ServerProxy实现，同样先获取队头指针和队尾指针，然后计算可读数据，将可读数据转给传入的buffer。拷贝完数据，然后更新读数据指针，然后同步通知Client端，唤醒挂起的进程。
```cpp
// frameworks/av/media/libaudioclient/AudioTrackShared.cpp
__attribute__((no_sanitize("integer")))
status_t ServerProxy::obtainBuffer(Buffer* buffer, bool ackFlush)
{
    LOG_ALWAYS_FATAL_IF(buffer == NULL || buffer->mFrameCount == 0,
            "%s: null or zero frame buffer, buffer:%p", __func__, buffer);
    if (mIsShutdown) {
        goto no_init;
    }
    {
    audio_track_cblk_t* cblk = mCblk;
    // compute number of frames available to write (AudioTrack) or read (AudioRecord),
    // or use previous cached value from framesReady(), with added barrier if it omits.
    int32_t front;
    int32_t rear;
    // See notes on barriers at ClientProxy::obtainBuffer()
    // 注意使用带内存屏障的函数android_atomic_acquire_load
    // 获取队头和队尾
    if (mIsOut) {
        flushBufferIfNeeded(); // might modify mFront
        rear = android_atomic_acquire_load(&cblk->u.mStreaming.mRear);
        front = cblk->u.mStreaming.mFront;
    } else {
        front = android_atomic_acquire_load(&cblk->u.mStreaming.mFront);
        rear = cblk->u.mStreaming.mRear;
    }
    ssize_t filled = rear - front;
    // pipe should not already be overfull
    if (!(0 <= filled && (size_t) filled <= mFrameCount)) {
        ALOGE("Shared memory control block is corrupt (filled=%zd, mFrameCount=%zu); 
               shutting down", filled, mFrameCount);
        mIsShutdown = true;
    }
    if (mIsShutdown) {
        goto no_init;
    }
    // don't allow filling pipe beyond the nominal size
    size_t availToServer;
    if (mIsOut) {
        availToServer = filled;
        mAvailToClient = mFrameCount - filled;
    } else {
        availToServer = mFrameCount - filled;
        mAvailToClient = filled;
    }
    // 'availToServer' may be non-contiguous, so return only the first contiguous chunk
    size_t part1;
    if (mIsOut) {
        front &= mFrameCountP2 - 1;
        part1 = mFrameCountP2 - front;
    } else {
        rear &= mFrameCountP2 - 1;
        part1 = mFrameCountP2 - rear;
    }
    if (part1 > availToServer) {
        part1 = availToServer;
    }
    size_t ask = buffer->mFrameCount;
    if (part1 > ask) {
        part1 = ask;
    }
    // is assignment redundant in some cases?
    buffer->mFrameCount = part1;
    buffer->mRaw = part1 > 0 ?
            &((char *) mBuffers)[(mIsOut ? front : rear) * mFrameSize] : NULL;
    buffer->mNonContig = availToServer - part1;
    // After flush(), allow releaseBuffer() on a previously obtained buffer;
    // see "Acknowledge any pending flush()" in audioflinger/Tracks.cpp.
    if (!ackFlush) {
        mUnreleased = part1;
    }
    return part1 > 0 ? NO_ERROR : WOULD_BLOCK;
    }
no_init:
    buffer->mFrameCount = 0;
    buffer->mRaw = NULL;
    buffer->mNonContig = 0;
    mUnreleased = 0;
    return NO_INIT;
}

__attribute__((no_sanitize("integer")))
void ServerProxy::releaseBuffer(Buffer* buffer)
{
    ..... // 条件检查
    mUnreleased -= stepCount;
    audio_track_cblk_t* cblk = mCblk;
    // 释放buffer控制权，其实是移动队头（队尾），更新读数据位置，注意内存屏障的使用
    if (mIsOut) {
        int32_t front = cblk->u.mStreaming.mFront;
        android_atomic_release_store(stepCount + front, &cblk->u.mStreaming.mFront);
    } else {
        int32_t rear = cblk->u.mStreaming.mRear;
        android_atomic_release_store(stepCount + rear, &cblk->u.mStreaming.mRear);
    }
    cblk->mServer += stepCount;
    mReleased += stepCount;
    size_t half = mFrameCount / 2;
    if (half == 0) {
        half = 1;
    }
    size_t minimum = (size_t) cblk->mMinimum;
    if (minimum == 0) {
        minimum = mIsOut ? half : 1;
    } else if (minimum > half) {
        minimum = half;
    }
    // FIXME AudioRecord wakeup needs to be optimized; it currently wakes up client every time
    if (!mIsOut || (mAvailToClient + stepCount >= minimum)) {
        int32_t old = android_atomic_or(CBLK_FUTEX_WAKE, &cblk->mFutex);
        if (!(old & CBLK_FUTEX_WAKE)) {
            // 同步控制，up (V)操作,原子性的给cblk->mFutex同步变量加1，
            // FUTEX_WAKE，原子性的检查cblk->mFutex中计数器的值是否为正值,
            // 如果不是则唤醒一个或者多个等待进程。
            (void) syscall(__NR_futex, &cblk->mFutex,
                    mClientInServer ? FUTEX_WAKE_PRIVATE : FUTEX_WAKE, 1);
        }
    }
    buffer->mFrameCount = 0;
    buffer->mRaw = NULL;
    buffer->mNonContig = 0;
}
```