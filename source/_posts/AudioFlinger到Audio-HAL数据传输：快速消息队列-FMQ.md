---
title: AudioFlinger到Audio HAL数据传输：快速消息队列(FMQ)
date: 2018-01-25 21:05:46
tags: Android
categories: Android Audio HAL
---
Android 8.0为了降低耦合度，开始独立HAL层，对于Audio，增加进程audio-hal-2-0，HAL的so不在和audioserver处于同一进程，而位于audio-hal-2-0进程中，这样从native层将数据传输到HAL层，就属于跨进程传输，这就需要考虑效率问题。HIDL的远程过程调用 (RPC) 基础架构使用Binder机制，这意味着调用涉及开销、需要内核操作，并且可以触发调度程序操作。不过，好在Android提供了快速消息队列 (FMQ) ，可以对于必须在开销较小且无内核参与的进程之间传输数据的情况。
## 获取IDevicesFactory远程代理
前面在Audio HAL server启动时会创建IDevicesFactory服务，在AudioFlinger创建时会创建DevicesFactoryHalHidl，则会获取IDevicesFactory的远程代理。
```cpp
// frameworks/av/media/libaudiohal/DeviceFactoryHalHidl.cpp
DevicesFactoryHalHidl::DevicesFactoryHalHidl() {
    mDevicesFactory = IDevicesFactory::getService();
    if (mDevicesFactory != 0) {
        // It is assumed that DevicesFactory is owned by AudioFlinger
        // and thus have the same lifespan.
        mDevicesFactory->linkToDeath(HalDeathHandler::getInstance(), 0 /*cookie*/);
    } else {
        ALOGE("Failed to obtain IDevicesFactory service, terminating process.");
        exit(1);
    }
}
```
下面是IDevicesFactory::getService函数实现，我们在Audio HAL server启动时也研究过，当时参数getStub为true，所以前半段获取代理的部分不会执行，而在获取代理时，后半段创建BsDevicesFactory就不会执行了。同样首先获取ServiceManager的代理BpHwServiceManager,然后通过BpHwServiceManager获取Transport类型，最后获取BpHwDevicesFactory。
```cpp
// out/soong/.intermediates/hardware/interfaces/audio/2.0/ \
// android.hardware.audio@2.0_genc++/gen/android/hardware/audio/2.0/DevicesFactoryAll.cpp
::android::sp<IDevicesFactory> IDevicesFactory::getService(
                               const std::string &serviceName, const bool getStub) {
    using ::android::hardware::defaultServiceManager;
    using ::android::hardware::details::waitForHwService;
    using ::android::hardware::getPassthroughServiceManager;
    using ::android::hardware::Return;
    using ::android::sp;
    using Transport = ::android::hidl::manager::V1_0::IServiceManager::Transport;

    sp<IDevicesFactory> iface = nullptr;

    const sp<::android::hidl::manager::V1_0::IServiceManager> sm = defaultServiceManager();
    if (sm == nullptr) {
        ALOGE("getService: defaultServiceManager() is null");
        return nullptr;
    }

    Return<Transport> transportRet = sm->getTransport(IDevicesFactory::descriptor, serviceName);

    if (!transportRet.isOk()) {
        ALOGE("getService: defaultServiceManager()->getTransport returns %s", 
                                             transportRet.description().c_str());
        return nullptr;
    }
    Transport transport = transportRet;
    const bool vintfHwbinder = (transport == Transport::HWBINDER);
    const bool vintfPassthru = (transport == Transport::PASSTHROUGH);

    ......

    for (int tries = 0; !getStub && (vintfHwbinder || (vintfLegacy && tries == 0)); tries++) {
        if (tries > 1) {
            ALOGI("getService: Will do try %d for %s/%s in 1s...", 
                         tries, IDevicesFactory::descriptor, serviceName.c_str());
            sleep(1);
        }
        if (vintfHwbinder && tries > 0) {
            waitForHwService(IDevicesFactory::descriptor, serviceName);
        }
        Return<sp<::android::hidl::base::V1_0::IBase>> ret = 
                sm->get(IDevicesFactory::descriptor, serviceName);
        if (!ret.isOk()) {
            ALOGE("IDevicesFactory: defaultServiceManager()->get returns %s",
                                                       ret.description().c_str());
            break;
        }
        sp<::android::hidl::base::V1_0::IBase> base = ret;
        if (base == nullptr) {
            if (tries > 0) {
                ALOGW("IDevicesFactory: found null hwbinder interface");
            }continue;
        }
        Return<sp<IDevicesFactory>> castRet = IDevicesFactory::castFrom(base, 
                                                           true /* emitError */);
        if (!castRet.isOk()) {
            if (castRet.isDeadObject()) {
                ALOGW("IDevicesFactory: found dead hwbinder service");
                continue;
            } else {
                ALOGW("IDevicesFactory: cannot call into hwbinder service: %s; No permission? 
                      Check for selinux denials.", castRet.description().c_str());
                break;
            }
        }
        iface = castRet;
        if (iface == nullptr) {
            ALOGW("IDevicesFactory: received incompatible service; bug in hwservicemanager?");
            break;
        }
        return iface;
    }
    ......
}
```
Transport的类型虽然从ServiceManager获取，但是其实是配置在system/manifest.xml（源文件一般在system/libhidl目录下）或vendor/manifest.xml（源文件一般在device/qcom子目录中）中,如下是Audio HAL部分，起transport类型为hwbinder。
```xml
<manifest version="1.0" type="device">
    <hal format="hidl">
        <name>android.hardware.audio</name>
        <transport>hwbinder</transport>
        <version>2.0</version>
        <interface>
            <name>IDevicesFactory</name>
            <instance>default</instance>
        </interface>
    </hal>
    ......
</manifest>
```
接下来进入BpHwServiceManager的get函数，将IDevicesFactory::descriptor和serviceName传入BpHwServiceManager，BpHwServiceManager将其封装在Parcel类型_hidl_data，然后通过remote()函数，利用Binder驱动将_hidl_data传递给BnHwServiceManager。BnHwServiceManager收到_hidl_data数据，从中还原descriptor和serviceName，然后调用_hidl_mImpl(BsServiceManager类型)的get函数，最终从ServiceManager（system/hwservicemanager/ServiceManager.cpp）中返回BsDevicesFactory。然后通过toBinder函数将其转化为Binder对象BnHwDevicesFactory，返回到BpHwServiceManager执行fromBinder函数，因为BnHwDevicesFactory间接继承自BHwBinder，所以localBinder()返回不为空，即创建BpHwBase封装BnHwDevicesFactory返回。
```cpp
// out/soong/.intermediates/system/libhidl/transport/manager/1.0/ \
// android.hidl.manager@1.0_genc++/gen/android/hidl/manager/1.0/ServiceManagerAll.cpp
::android::hardware::Return<::android::sp<::android::hidl::base::V1_0::IBase>>
                       BpHwServiceManager::get(
                          const ::android::hardware::hidl_string& fqName, 
                          const ::android::hardware::hidl_string& name) {
    atrace_begin(ATRACE_TAG_HAL, "HIDL::IServiceManager::get::client");
    ......_

    ::android::hardware::Parcel _hidl_data;
    ::android::hardware::Parcel _hidl_reply;
    ::android::status_t _hidl_err;
    ::android::hardware::Status _hidl_status;

    ::android::sp<::android::hidl::base::V1_0::IBase> _hidl_out_service;

    _hidl_err = _hidl_data.writeInterfaceToken(IServiceManager::descriptor);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    size_t _hidl_fqName_parent;

    _hidl_err = _hidl_data.writeBuffer(&fqName, sizeof(fqName), &_hidl_fqName_parent);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = ::android::hardware::writeEmbeddedToParcel(
            fqName,
            &_hidl_data,
            _hidl_fqName_parent,
            0 /* parentOffset */);

    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    size_t _hidl_name_parent;

    _hidl_err = _hidl_data.writeBuffer(&name, sizeof(name), &_hidl_name_parent);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = ::android::hardware::writeEmbeddedToParcel(
            name,
            &_hidl_data,
            _hidl_name_parent,
            0 /* parentOffset */);

    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = remote()->transact(1 /* get */, _hidl_data, &_hidl_reply);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = ::android::hardware::readFromParcel(&_hidl_status, _hidl_reply);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    if (!_hidl_status.isOk()) { return _hidl_status; }

    {
        ::android::sp<::android::hardware::IBinder> _hidl__hidl_out_service_binder;
        _hidl_err = _hidl_reply.readNullableStrongBinder(&_hidl__hidl_out_service_binder);
        if (_hidl_err != ::android::OK) { goto _hidl_error; }

        _hidl_out_service = ::android::hardware::fromBinder<
                            ::android::hidl::base::V1_0::IBase,
                            ::android::hidl::base::V1_0::BpHwBase,
                            ::android::hidl::base::V1_0::BnHwBase>
                            (_hidl__hidl_out_service_binder);
    }

    atrace_end(ATRACE_TAG_HAL);
    ......

    _hidl_status.setFromStatusT(_hidl_err);
    return ::android::hardware::Return<::android::
             sp<::android::hidl::base::V1_0::IBase>>(_hidl_out_service);

_hidl_error:
    _hidl_status.setFromStatusT(_hidl_err);
    return ::android::hardware::Return<::android::
             sp<::android::hidl::base::V1_0::IBase>>(_hidl_status);
}

::android::status_t BnHwServiceManager::onTransact(
        uint32_t _hidl_code,
        const ::android::hardware::Parcel &_hidl_data,
        ::android::hardware::Parcel *_hidl_reply,
        uint32_t _hidl_flags,
        TransactCallback _hidl_cb) {
    ::android::status_t _hidl_err = ::android::OK;

    switch (_hidl_code) {
        case 1 /* get */:
        {
            if (!_hidl_data.enforceInterface(IServiceManager::descriptor)) {
                _hidl_err = ::android::BAD_TYPE;
                break;
            }

            const ::android::hardware::hidl_string* fqName;
            const ::android::hardware::hidl_string* name;

            size_t _hidl_fqName_parent;

            _hidl_err = _hidl_data.readBuffer(sizeof(*fqName), &_hidl_fqName_parent,
                                          reinterpret_cast<const void **>(&fqName));

            if (_hidl_err != ::android::OK) { break; }

            _hidl_err = ::android::hardware::readEmbeddedFromParcel(
                    const_cast<::android::hardware::hidl_string &>(*fqName),
                    _hidl_data,
                    _hidl_fqName_parent,
                    0 /* parentOffset */);

            if (_hidl_err != ::android::OK) { break; }

            size_t _hidl_name_parent;

            _hidl_err = _hidl_data.readBuffer(sizeof(*name), 
                        &_hidl_name_parent,  reinterpret_cast<const void **>(&name));

            if (_hidl_err != ::android::OK) { break; }

            _hidl_err = ::android::hardware::readEmbeddedFromParcel(
                    const_cast<::android::hardware::hidl_string &>(*name),
                    _hidl_data,
                    _hidl_name_parent,
                    0 /* parentOffset */);

            if (_hidl_err != ::android::OK) { break; }

            atrace_begin(ATRACE_TAG_HAL, "HIDL::IServiceManager::get::server");
            ......

            ::android::sp<::android::hidl::base::V1_0::IBase> _hidl_out_service = 
                                                  _hidl_mImpl->get(*fqName, *name);

            ::android::hardware::writeToParcel(::android::hardware::Status::ok(), _hidl_reply);

            if (_hidl_out_service == nullptr) {
                _hidl_err = _hidl_reply->writeStrongBinder(nullptr);
            } else {
                ::android::sp<::android::hardware::IBinder> _hidl_binder = 
                                       ::android::hardware::toBinder<
                                       ::android::hidl::base::V1_0::IBase, 
                                       ::android::hidl::base::V1_0::BpHwBase>
                                       (_hidl_out_service);
                if (_hidl_binder.get() != nullptr) {
                    _hidl_err = _hidl_reply->writeStrongBinder(_hidl_binder);
                } else {
                    _hidl_err = ::android::UNKNOWN_ERROR;
                }
            }
            /* _hidl_err ignored! */

            atrace_end(ATRACE_TAG_HAL);
            ......

            _hidl_cb(*_hidl_reply);
            break;
        }
    }

    .....
}
```
最后将BpHwBase转化为代理对象BpHwDevicesFactory。调用IDevicesFactory::castFrom函数，继而调用castInterface函数，BpHwBase的isRemote()返回true，我们看到对于binderized mode，会创建BpHwDevicesFactory对象。
```cpp
// out/soong/.intermediates/hardware/interfaces/audio/2.0/ \
// android.hardware.audio@2.0_genc++/gen/android/hardware/audio/2.0/DevicesFactoryAll.cpp
::android::hardware::Return<::android::sp<IDevicesFactory>> IDevicesFactory::castFrom(
              const ::android::sp<::android::hidl::base::V1_0::IBase>& parent, 
              bool emitError) {
    return ::android::hardware::details::castInterface<
                                      IDevicesFactory, 
                                      ::android::hidl::base::V1_0::IBase, 
                                      BpHwDevicesFactory, 
                                      ::android::hidl::base::V1_0::BpHwBase>(
            parent, "android.hardware.audio@2.0::IDevicesFactory", emitError);
}

// system/libhidl/transport/include/hidl/HidlTransportSupport.h
// cast the interface IParent to IChild.
// Return nonnull if cast successful.
// Return nullptr if:
// 1. parent is null
// 2. cast failed because IChild is not a child type of IParent.
// 3. !emitError, calling into parent fails.
// Return an error Return object if:
// 1. emitError, calling into parent fails.
template<typename IChild, typename IParent, typename BpChild, typename BpParent>
Return<sp<IChild>> castInterface(sp<IParent> parent, 
                                 const char *childIndicator, 
                                 bool emitError) {
    if (parent.get() == nullptr) {
        // casts always succeed with nullptrs.
        return nullptr;
    }
    Return<bool> canCastRet = details::canCastInterface(parent.get(), childIndicator, emitError);
    if (!canCastRet.isOk()) {
        // call fails, propagate the error if emitError
        return emitError
                ? details::StatusOf<bool, sp<IChild>>(canCastRet)
                : Return<sp<IChild>>(sp<IChild>(nullptr));
    }

    if (!canCastRet) {
        return sp<IChild>(nullptr); // cast failed.
    }
    // TODO b/32001926 Needs to be fixed for socket mode.
    if (parent->isRemote()) {
        // binderized mode. Got BpChild. grab the remote and wrap it.
        return sp<IChild>(new BpChild(toBinder<IParent, BpParent>(parent)));
    }
    // Passthrough mode. Got BnChild and BsChild.
    return sp<IChild>(static_cast<IChild *>(parent.get()));
}
```
通过如上系列流程，则得到了代理对象BpHwDevicesFactory，接下来就会通过这个代理对象和HAL通信了。
## Audio Device的加载
在AudioPolicyService启动过程中，我们知道会加载Audio Device,在AudioFlinger中会openDevice，将返回的DeviceHalHidl（DeviceHalInterface的子类）的用AudioHwDevice封装。
```cpp
// frameworks/av/service/audioflinger/AudioFlinger.cpp
// loadHwModule_l() must be called with AudioFlinger::mLock held
audio_module_handle_t AudioFlinger::loadHwModule_l(const char *name)
{
    ......
    sp<DeviceHalInterface> dev;
    int rc = mDevicesFactoryHal->openDevice(name, &dev);
    ......
    audio_module_handle_t handle = 
               (audio_module_handle_t) nextUniqueId(AUDIO_UNIQUE_ID_USE_MODULE);
    mAudioHwDevs.add(handle, new AudioHwDevice(handle, name, dev, flags));
    ALOGI("loadHwModule() Loaded %s audio interface, handle %d", name, handle);
    return handle;
}
```
mDevicesFactory就是我们获取的BpHwDevicesFactory代理对象，mDevicesFactory->openDevice会获得IDevice代理对象，DeviceHalHidl会封装IDevice代理对象。
```cpp
// frameworks/av/media/libaudiohal/DevicesFactoryHalHidl.cpp
status_t DevicesFactoryHalHidl::openDevice(const char *name, sp<DeviceHalInterface> *device) {
    if (mDevicesFactory == 0) return NO_INIT;
    IDevicesFactory::Device hidlDevice;
    status_t status = nameFromHal(name, &hidlDevice);
    if (status != OK) return status;
    Result retval = Result::NOT_INITIALIZED;
    Return<void> ret = mDevicesFactory->openDevice(
            hidlDevice,
            [&](Result r, const sp<IDevice>& result) {
                retval = r;
                if (retval == Result::OK) {
                    *device = new DeviceHalHidl(result);
                }
            });
    if (ret.isOk()) {
        if (retval == Result::OK) return OK;
        else if (retval == Result::INVALID_ARGUMENTS) return BAD_VALUE;
        else return NO_INIT;
    }
    return FAILED_TRANSACTION;
}
```
下面会通过BpHwDevicesFactory代理对象以Parcel类型_hidl_data封装需求，通过Binder驱动传递给BnHwDevicesFactory。
```cpp
// out/soong/.intermediates/hardware/interfaces/audio/2.0/ \
// android.hardware.audio@2.0_genc++/gen/android/hardware/audio/2.0/DevicesFactoryAll.cpp
::android::hardware::Return<void> BpHwDevicesFactory::openDevice(
                         IDevicesFactory::Device device, openDevice_cb _hidl_cb) {
    if (_hidl_cb == nullptr) {
        return ::android::hardware::Status::fromExceptionCode(
                ::android::hardware::Status::EX_ILLEGAL_ARGUMENT);
    }

    atrace_begin(ATRACE_TAG_HAL, "HIDL::IDevicesFactory::openDevice::client");
    ......

    ::android::hardware::Parcel _hidl_data;
    ::android::hardware::Parcel _hidl_reply;
    ::android::status_t _hidl_err;
    ::android::hardware::Status _hidl_status;

    Result _hidl_out_retval;
    ::android::sp<IDevice> _hidl_out_result;

    _hidl_err = _hidl_data.writeInterfaceToken(IDevicesFactory::descriptor);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = _hidl_data.writeInt32((int32_t)device);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = remote()->transact(1 /* openDevice */, _hidl_data, &_hidl_reply);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = ::android::hardware::readFromParcel(&_hidl_status, _hidl_reply);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    if (!_hidl_status.isOk()) { return _hidl_status; }

    _hidl_err = _hidl_reply.readInt32((int32_t *)&_hidl_out_retval);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    {
        ::android::sp<::android::hardware::IBinder> _hidl__hidl_out_result_binder;
        _hidl_err = _hidl_reply.readNullableStrongBinder(&_hidl__hidl_out_result_binder);
        if (_hidl_err != ::android::OK) { goto _hidl_error; }

        _hidl_out_result = ::android::hardware::fromBinder<IDevice,BpHwDevice,BnHwDevice>
                                                (_hidl__hidl_out_result_binder);
    }

    _hidl_cb(_hidl_out_retval, _hidl_out_result);

    atrace_end(ATRACE_TAG_HAL);
    ......

    _hidl_status.setFromStatusT(_hidl_err);
    return ::android::hardware::Return<void>();

_hidl_error:
    _hidl_status.setFromStatusT(_hidl_err);
    return ::android::hardware::Return<void>(_hidl_status);
}

```
需求到达BnHwDevicesFactory，BnHwDevicesFactory会调用BsDevicesFactory的openDevice的函数，将返回值通过toBinder函数转化为Binder对象BnHwPrimaryDevice。同样由于BnHwPrimaryDevice的localBinder()返回不为空，返回到BpHwDevicesFactory，通过fromBinder函数将Binder对象BnHwPrimaryDevice转化为BpHwPrimaryDevice。
```cpp
// out/soong/.intermediates/hardware/interfaces/audio/2.0/ \
// android.hardware.audio@2.0_genc++/gen/android/hardware/audio/2.0/DevicesFactoryAll.cpp
::android::status_t BnHwDevicesFactory::onTransact(
        uint32_t _hidl_code,
        const ::android::hardware::Parcel &_hidl_data,
        ::android::hardware::Parcel *_hidl_reply,
        uint32_t _hidl_flags,
        TransactCallback _hidl_cb) {
    ::android::status_t _hidl_err = ::android::OK;

    switch (_hidl_code) {
        case 1 /* openDevice */:
        {
            if (!_hidl_data.enforceInterface(IDevicesFactory::descriptor)) {
                _hidl_err = ::android::BAD_TYPE;
                break;
            }

            IDevicesFactory::Device device;

            _hidl_err = _hidl_data.readInt32((int32_t *)&device);
            if (_hidl_err != ::android::OK) { break; }

            atrace_begin(ATRACE_TAG_HAL, "HIDL::IDevicesFactory::openDevice::server");
            ......

            bool _hidl_callbackCalled = false;

            _hidl_mImpl->openDevice(device, 
                [&](const auto &_hidl_out_retval, const auto &_hidl_out_result) {

                if (_hidl_callbackCalled) {
                    LOG_ALWAYS_FATAL("openDevice: _hidl_cb called a second time, 
                                                        but must be called once.");
                }
                _hidl_callbackCalled = true;

                ::android::hardware::writeToParcel(
                               ::android::hardware::Status::ok(), _hidl_reply);

                _hidl_err = _hidl_reply->writeInt32((int32_t)_hidl_out_retval);
                /* _hidl_err ignored! */

                if (_hidl_out_result == nullptr) {
                    _hidl_err = _hidl_reply->writeStrongBinder(nullptr);
                } else {
                    ::android::sp<::android::hardware::IBinder> _hidl_binder = 
                            ::android::hardware::toBinder<
                            IDevice, BpHwDevice>(_hidl_out_result);
                    if (_hidl_binder.get() != nullptr) {
                        _hidl_err = _hidl_reply->writeStrongBinder(_hidl_binder);
                    } else {
                        _hidl_err = ::android::UNKNOWN_ERROR;
                    }
                }
                /* _hidl_err ignored! */

                atrace_end(ATRACE_TAG_HAL);
                ......

                _hidl_cb(*_hidl_reply);
            });

            if (!_hidl_callbackCalled) {
                LOG_ALWAYS_FATAL("openDevice: _hidl_cb not called, but must be called once.");
            }

            break;
        }
    }

    ......
}
```
在BsDevicesFactory中最终调用DevicesFactory的openDevice函数，创建PrimaryDevice对象返回到BsDevicesFactory，因为PrimaryDevice的isRemote函数会返回false，所以会执行wrapPassthrough函数，返回BsPrimaryDevice。
```cpp
// out/soong/.intermediates/hardware/interfaces/audio/2.0/ \
// android.hardware.audio@2.0_genc++/gen/android/hardware/audio/2.0/BsDevicesFactory.h
struct BsDevicesFactory : IDevicesFactory, ::android::hardware::details::HidlInstrumentor {
    explicit BsDevicesFactory(const ::android::sp<IDevicesFactory> impl);
    // Methods from IDevicesFactory follow.
    ::android::hardware::Return<void> openDevice(
                IDevicesFactory::Device device, openDevice_cb _hidl_cb) {
        if (_hidl_cb == nullptr) {
            return ::android::hardware::Status::fromExceptionCode(
                    ::android::hardware::Status::EX_ILLEGAL_ARGUMENT);
        }

        atrace_begin(ATRACE_TAG_HAL, "HIDL::IDevicesFactory::openDevice::passthrough");
        ......

        auto _hidl_error = ::android::hardware::Void();
        auto _hidl_return = mImpl->openDevice(device, 
            [&](const auto &_hidl_out_retval, const auto &_hidl_out_result) {

            atrace_end(ATRACE_TAG_HAL);
            ......

            ::android::sp<IDevice> _hidl_out_wrapped_result;
            if (_hidl_out_result != nullptr && !_hidl_out_result->isRemote()) {
                _hidl_out_wrapped_result = IDevice::castFrom(::android::hardware::details::
                                  wrapPassthrough<IDevice>(_hidl_out_result));
                if (_hidl_out_wrapped_result == nullptr) {
                    _hidl_error = ::android::hardware::Status::fromExceptionCode(
                            ::android::hardware::Status::EX_TRANSACTION_FAILED,
                            "Cannot wrap passthrough interface.");
                    return;
                }
            } else {
                _hidl_out_wrapped_result = _hidl_out_result;
            }

            _hidl_cb(_hidl_out_retval, _hidl_out_wrapped_result);
        });

        return _hidl_return;
    }
    
    ......
private:
    const ::android::sp<IDevicesFactory> mImpl;
    ......
};
```
然后执行IDevice::castFrom函数，这里会返回BsPrimaryDevice的基类。
```
// out/soong/.intermediates/hardware/interfaces/audio/2.0/ \
// android.hardware.audio@2.0_genc++/gen/android/hardware/audio/2.0/DeviceAll.cpp
::android::hardware::Return<::android::sp<IDevice>> IDevice::castFrom
                      (const ::android::sp<IDevice>& parent, bool /* emitError */) {
    return parent;
}
```
wrapPassthrough函数关键在于从预先创建的gBsConstructorMap中获取BsPrimaryDevice，进而封装PrimaryDevice。
```cpp
// system/libhidl/transport/include/hidl/HidlPassThroughSupport.h
/*
 * Wrap the given interface with the smallest BsChild possible. Will return the
 * argument directly if nullptr or isRemote().
 */
template<typename IType>
sp<::android::hidl::base::V1_0::IBase> wrapPassthrough(
        sp<IType> iface) {
    if (iface.get() == nullptr || iface->isRemote()) {
        // doesn't know how to handle it.
        return iface;
    }
    std::string myDescriptor = getDescriptor(iface.get());
    if (myDescriptor.empty()) {
        // interfaceDescriptor fails
        return nullptr;
    }
    auto func = gBsConstructorMap.get(myDescriptor, nullptr);
    if (!func) {
        return nullptr;
    }
    return func(static_cast<void *>(iface.get()));
}
```
DevicesFactory是IDevicesFactory终极服务端实现，在openDevice中创建PrimaryDevice或Device封装audio hal的设备audio_hw_device_t。
```cpp
// hardware/interface/audio/2.0/default/DevicesFactory.cpp
Return<void> DevicesFactory::openDevice(IDevicesFactory::Device device, openDevice_cb _hidl_cb) {
    audio_hw_device_t *halDevice;
    Result retval(Result::INVALID_ARGUMENTS);
    sp<IDevice> result;
    const char* moduleName = deviceToString(device);
    if (moduleName != nullptr) {
        int halStatus = loadAudioInterface(moduleName, &halDevice);
        if (halStatus == OK) {
            if (device == IDevicesFactory::Device::PRIMARY) {
                result = new PrimaryDevice(halDevice);
            } else {
                result = new ::android::hardware::audio::V2_0::implementation::
                    Device(halDevice);
            }
            retval = Result::OK;
        } else if (halStatus == -EINVAL) {
            retval = Result::NOT_INITIALIZED;
        }
    }
    _hidl_cb(retval, result);
    return Void();
}
```
## Audio Stream的加载
创建Audio Stream也是在AudioPolicyService启动过程中。从AudioFlinger::openOutput_l到AudioHwDevice::openOutputStream到AudioStreamOut::open到DeviceHalHidl::openOutputStream，经过这个流程最后会使用BpHwPrimaryDevice创建Audio Stream，然后以其为参数创建PlaybackThread，这样后续播放数据时，就会通过Audio Stream将数据传递到Audio HAL。
```cpp
// frameworks/av/service/audioflinger/AudioFlinger.cpp
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

    ...... // 以outputStream为参数创建PlaybackThread

    return 0;
}

// frameworks/av/service/audioflinger/AudioHwDevice.cpp
status_t AudioHwDevice::openOutputStream(
        AudioStreamOut **ppStreamOut,
        audio_io_handle_t handle,
        audio_devices_t devices,
        audio_output_flags_t flags,
        struct audio_config *config,
        const char *address)
{

    struct audio_config originalConfig = *config;
    AudioStreamOut *outputStream = new AudioStreamOut(this, flags);

    // Try to open the HAL first using the current format.
    ALOGV("openOutputStream(), try "
            " sampleRate %d, Format %#x, "
            "channelMask %#x",
            config->sample_rate,
            config->format,
            config->channel_mask);
    status_t status = outputStream->open(handle, devices, config, address);

    ......
}

// frameworks/av/service/audioflinger/AudioStreamOut.cpp
status_t AudioStreamOut::open(
        audio_io_handle_t handle,
        audio_devices_t devices,
        struct audio_config *config,
        const char *address)
{
    sp<StreamOutHalInterface> outStream;

    audio_output_flags_t customFlags = (config->format == AUDIO_FORMAT_IEC61937)
                ? (audio_output_flags_t)(flags | AUDIO_OUTPUT_FLAG_IEC958_NONAUDIO)
                : flags;

    int status = hwDev()->openOutputStream(
            handle,
            devices,
            customFlags,
            config,
            address,
            &outStream);
    ALOGV("AudioStreamOut::open(), HAL returned "
            " stream %p, sampleRate %d, Format %#x, "
            "channelMask %#x, status %d",
            outStream.get(),
            config->sample_rate,
            config->format,
            config->channel_mask,
            status);

    ......
    return status;
}

// frameworks/av/media/libaudiohal/DeviceHalHidl.cpp
status_t DeviceHalHidl::openOutputStream(
        audio_io_handle_t handle,
        audio_devices_t devices,
        audio_output_flags_t flags,
        struct audio_config *config,
        const char *address,
        sp<StreamOutHalInterface> *outStream) {
    if (mDevice == 0) return NO_INIT;
    DeviceAddress hidlDevice;
    status_t status = deviceAddressFromHal(devices, address, &hidlDevice);
    if (status != OK) return status;
    AudioConfig hidlConfig;
    HidlUtils::audioConfigFromHal(*config, &hidlConfig);
    Result retval = Result::NOT_INITIALIZED;
    Return<void> ret = mDevice->openOutputStream(
            handle,
            hidlDevice,
            hidlConfig,
            AudioOutputFlag(flags),
            [&](Result r, const sp<IStreamOut>& result, const AudioConfig& suggestedConfig) {
                retval = r;
                if (retval == Result::OK) {
                    *outStream = new StreamOutHalHidl(result);
                }
                HidlUtils::audioConfigToHal(suggestedConfig, config);
            });
    return processReturn("openOutputStream", ret, retval);
}
```
接下来会类似获取BpHwPrimaryDevice一样，获取BpHwStreamOut，将其封装在StreamOutHalHidl中，后续通信通过BpHwStreamOut和BnHwStreamOut（封装BsStreamOut和StreamOut）。
## 读写数据流程
```cpp
// frameworks/av/service/audioflinger/Threads.cpp
AudioFlinger::PlaybackThread::PlaybackThread(const sp<AudioFlinger>& audioFlinger,
                                             AudioStreamOut* output,
                                             audio_io_handle_t id,
                                             audio_devices_t device,
                                             type_t type,
                                             bool systemReady)
    :   ThreadBase(audioFlinger, id, device, AUDIO_DEVICE_NONE, type, systemReady),
        ......
        mOutput(output),
        ......
{
}
```

## FMQ的实现
