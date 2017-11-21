---
title: Android Audio Server启动
date: 2017-11-08 10:44:20
tags: Android
categories: Android Audio
---

Android audioserver是Audio系统native的服务，也是连接Audio Framework和Audio HAL的一个纽带，其中包含了AudioFlinger、AudioPolicyService、AAudioService、RadioService、SoundTriggerHwService等服务。源代码位于frameworks/av/media/audioserver

从Android 7.0开始，Audio相关的service从mediaserver中转移到audioserver，把audio，camera及mediaplayerservice做了一个拆分，这样不会显得臃肿、职能更加独立且安全性更高。拆分之后audioserver还是一个native service，还是从init进程中启动，如下是其在audioserver.rc中的启动代码。

```sh
service audioserver /system/bin/audioserver
    class main                                   # audioserver和class main行为一致
    user audioserver                             # 用户归属，uid：AID_AUDIOSERVER
    # media gid needed for /dev/fm (radio) and for /data/misc/media (tee)
    group audio camera drmrpc inet media mediadrm net_bt \
          net_bt_admin net_bw_acct oem_2901                         # 用户组归属
    ioprio rt 4                                                     # io调度优先级
    # 当子进程被创建的时候，将子进程的pid写入到给定的文件中,cgroup/cpuset用法
    writepid /dev/cpuset/foreground/tasks /dev/stune/foreground/tasks                
    onrestart restart audio-hal-2-0                     # audioserver重启会重启hal
```
我们看到Android 8.0当重启audioserver时，会重启audio-hal-2-0，这个服务是audio hal的服务，在android 8.0中，framework native进程与hal分离，hal不在和framework native处于同一个进程，而是独立进程，进程间通过binder通信。先不讲HAL binder化，我们先看看audioserver中包含哪几个服务。
```cpp
int main(int argc __unused, char **argv)
{
   signal(SIGPIPE, SIG_IGN);

   bool doLog = (bool) property_get_bool("ro.test_harness", 0);

   pid_t childPid;

   ......

   if (doLog && (childPid = fork()) != 0) {

      ......

   } else {

      sp<ProcessState> proc(ProcessState::self());
      sp<IServiceManager> sm = defaultServiceManager();
      ALOGI("ServiceManager: %p", sm.get());
      AudioFlinger::instantiate();
      AudioPolicyService::instantiate();
      AAudioService::instantiate();
      RadioService::instantiate();
      SoundTriggerHwService::instantiate();

      ProcessState::self()->startThreadPool();

// FIXME: remove when BUG 31748996 is fixed
      android::hardware::ProcessState::self()->startThreadPool();

      IPCThreadState::self()->joinThreadPool();
   }
}
```
从如上代码可以看出，audioserver中回依次执行AudioFlinger、AudioPolicyService、AAudioService、RadioService、SoundTriggerHwService的instantiate函数。通过阅读源代码，由于继承的缘故这个五个service最终会调用BinderService的instantiate函数且将自己注册到ServiceManager中，后续client端可以通过service注册时用的name从ServiceManager返回server端。
```cpp
template<typename SERVICE>
class BinderService
{
public:
    static status_t publish(bool allowIsolated = false) {
        sp<IServiceManager> sm(defaultServiceManager());
        return sm->addService(
                String16(SERVICE::getServiceName()),
                new SERVICE(), allowIsolated);
    }

    static void publishAndJoinThreadPool(bool allowIsolated = false) {
        publish(allowIsolated);
        joinThreadPool();
    }

    static void instantiate() { publish(); }

    static status_t shutdown() { return NO_ERROR; }

private:
    static void joinThreadPool() {
        sp<ProcessState> ps(ProcessState::self());
        ps->startThreadPool();
        ps->giveThreadPoolName();
        IPCThreadState::self()->joinThreadPool();
    }
};
```
AudioFlinger（media.audio_flinger）：Audio系统的一个核心服务，是音频策略的执行者，负责输入输出流设备的管理及音频流数据的处理传输。

AudioPolicyService（media.audio_policy）：音频策略的制定者，负责音频设备切换的策略抉择、音量调节策略等。

AAudioService（media.aaudio）：这是Android 8.0加入角色，是OpenSL ES的另外一种选择，需要低延迟的高性能音频应用的另外一种选择。

RadioService（media.radio）：与FM相关的一个服务。

SoundTriggerHwService（media.sound_trigger_hw）：Android语音识别的native服务。
