---
title: Android Audio HAL server启动
date: 2017-11-28 10:41:54
tags: Android
categories: Android Audio HAL
---
我们知道硬件抽象层（HAL）是连接driver和native的桥梁，数据会从native层到HAL层，最终写入kernel。然而8.0之前的HAL和native处于同一进程，耦合度比较高。所以在Android 8.0，google对HAL做了重构，将HAL放在独立的进程，和native通过binder通信。下面我们就看看Android Audio HAL server的启动。
## Audio HAL server启动
同样Android Audio HAL server（audio-hal-2-0）从init进程启动，不过audio-hal-2-0的可执行文件在vendor分区，这也是为了降低耦合度，因为HAL一般由OEM/ODM实现，google不想HAL影响到Framework的更新，所以希望尽量将OEM/ODM的实现放到vendor分区，由这些厂商自己维护。
```sh
service audio-hal-2-0 /vendor/bin/hw/android.hardware.audio@2.0-service
    class hal                                   # audio-hal-2-0和class hal行为一致
    user audioserver                            # 用户归属，uid：AID_AUDIOSERVER
    # media gid needed for /dev/fm (radio) and for /data/misc/media (tee)
    group audio camera drmrpc inet media mediadrm net_bt \
                       net_bt_admin net_bw_acct # 用户组归属
    ioprio rt 4                                 # io调度优先级
    # 当子进程被创建的时候，将子进程的pid写入到给定的文件中,cgroup/cpuset用法
    writepid /dev/cpuset/foreground/tasks /dev/stune/foreground/tasks
    # audioflinger restarts itself when it loses connection with the hal
    # and its .rc file has an "onrestart restart audio-hal" rule, thus
    # an additional auto-restart from the init process isn't needed.
    oneshot                                     # 当此服务退出时不会自动重启
```
我们看到audio-hal-2-0的启动代码有IDevicesFactory，IEffectsFactory，ISoundTriggerHw及IBroadcastRadioFactory，这和native的audioserver相对应。且代码有一种熟悉的感觉，和audioserver的启动很相似，只是具体实现有些不一样，这里通过registerPassthroughServiceImplementation将如上四个服务注册到hwservicemanager，以供native调用。
```cpp
int main(int /* argc */, char* /* argv */ []) {
    configureRpcThreadpool(16, true /*callerWillJoin*/);
    android::status_t status;
    status = registerPassthroughServiceImplementation<IDevicesFactory>();
    LOG_ALWAYS_FATAL_IF(status != OK, "Error while registering audio service: %d", status);
    status = registerPassthroughServiceImplementation<IEffectsFactory>();
    LOG_ALWAYS_FATAL_IF(status != OK, "Error while registering audio effects 
                                                         service: %d", status);
    // Soundtrigger and FM radio might be not present.
    status = registerPassthroughServiceImplementation<ISoundTriggerHw>();
    ALOGE_IF(status != OK, "Error while registering soundtrigger service: %d", status);
    if (useBroadcastRadioFutureFeatures) {
        status = registerPassthroughServiceImplementation<
            broadcastradio::V1_1::IBroadcastRadioFactory>();
    } else {
        status = registerPassthroughServiceImplementation<
            broadcastradio::V1_0::IBroadcastRadioFactory>();
    }
    ALOGE_IF(status != OK, "Error while registering fm radio service: %d", status);
    joinRpcThreadpool();
    return status;
}
```
## IDevicesFactory注册到hwservicemanager
### __attribute__特性
在看registerPassthroughServiceImplementation函数之前，我们看看IDevicesFactory服务端创建时机，这里使用了GNU C的__attribute__机制，对于构造函数：
    __attribute__((constructor)) // 在main函数被调用之前调用
    __attribute__((destructor)) // 在main函数被调用之后调
如下代码发现，这里创建了两个全局变量，gBnConstructorMap存储了所有的服务端BnXXX,gBsConstructorMap存储了所有的服务端BsXXX，所以在进入main函数之前就创建了所有的BnXXX服务和BsXXX。
```cpp
// out/soong/.intermediates/hardware/interfaces/audio/2.0/ \
// android.hardware.audio@2.0_genc++/gen/android/hardware/audio/2.0/DevicesFactoryAll.cpp
const char* IDevicesFactory::descriptor("android.hardware.audio@2.0::IDevicesFactory");

__attribute__((constructor))static void static_constructor() {
    ::android::hardware::details::gBnConstructorMap.set(IDevicesFactory::descriptor,
            [](void *iIntf) -> ::android::sp<::android::hardware::IBinder> {
                return new BnHwDevicesFactory(static_cast<IDevicesFactory *>(iIntf));
            });
    ::android::hardware::details::gBsConstructorMap.set(IDevicesFactory::descriptor,
            [](void *iIntf) -> ::android::sp<::android::hidl::base::V1_0::IBase> {
                return new BsDevicesFactory(static_cast<IDevicesFactory *>(iIntf));
            });
};
```
这里还是用了C++ lambda表达式，开始的方括号称为lambda引导，它标志着lambda表达式的开始。后面的圆括号中的是lambda的参数列表，这与普通函数相同。这里只有一个形参*iIntf。注意，lambda的参数列表不允许指定形参的默认值，并且参数列表的长度是不可变的。大括号中的是lambda的主体，这里只有一条return语句，当然可以包含多条语句。当lambda表达式的主体是一条单一返回语句，而该语句在lambda表达式主体中返回一个值时，返回类型默认为返回值的类型。否则，返回void。这里指定了返回值类型为IBinder和IBase。
### IDevicesFactory注册到hwservicemanager流程
下面我们来看看是如何将服务BnXXX添加到hwservicemanager的。在如上的启动代码中我们看到，HAL服务端调用registerPassthroughServiceImplementation实现注册，这函数的实现在system/libhidl/transport/include/hidl/LegacySupport.h,这是一个模板方法，这里传入IDevicesFactory接口。
```cpp
/**
 * Registers passthrough service implementation.
 */
template<class Interface>
__attribute__((warn_unused_result))
status_t registerPassthroughServiceImplementation(
        std::string name = "default") {
    sp<Interface> service = Interface::getService(name, true /* getStub */);

    ......

    status_t status = service->registerAsService(name);
    
    ......

    return status;
}
```
所以首先会调用IDevicesFactory的getService方法。在IDevicesFactory.h中看到有getService方法的声明，在DevicesFactoryAll.cpp中有此方法的实现，最后会返回BsDevicesFactory实例。
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

    ......

    if (getStub || vintfPassthru || vintfLegacy) {
        const sp<::android::hidl::manager::V1_0::IServiceManager> pm = 
                                             getPassthroughServiceManager();
        if (pm != nullptr) {
            Return<sp<::android::hidl::base::V1_0::IBase>> ret = 
                    pm->get(IDevicesFactory::descriptor, serviceName);
            if (ret.isOk()) {
                sp<::android::hidl::base::V1_0::IBase> baseInterface = ret;
                if (baseInterface != nullptr) {
                    iface = new BsDevicesFactory(IDevicesFactory::castFrom(baseInterface));
                }
            }
        }
    }
    return iface;
}
```
这里getStub为true，所以前面一段代码不会执行，直接创建PassthroughServiceManager，PassthroughServiceManager的实现在system/libhidl/transport/ServiceManagement.cpp，然后执行其get方法。
```cpp
    Return<sp<IBase>> get(const hidl_string& fqName,
                          const hidl_string& name) override {
        std::string stdFqName(fqName.c_str());
        //fqName looks like android.hardware.foo@1.0::IFoo
        size_t idx = stdFqName.find("::");
        if (idx == std::string::npos ||
                idx + strlen("::") + 1 >= stdFqName.size()) {
            LOG(ERROR) << "Invalid interface name passthrough lookup: " << fqName;
            return nullptr;
        }
        // packageAndVersion为android.hardware.audio@2.0
        std::string packageAndVersion = stdFqName.substr(0, idx);

        // ifaceName为IDevicesFactory
        std::string ifaceName = stdFqName.substr(idx + strlen("::"));

        const std::string prefix = packageAndVersion + "-impl";
        const std::string sym = "HIDL_FETCH_" + ifaceName;
        const android_namespace_t* sphal_namespace = android_get_exported_namespace("sphal");
        const int dlMode = RTLD_LAZY;
        void *handle = nullptr;
        // TODO: lookup in VINTF instead
        // TODO(b/34135607): Remove HAL_LIBRARY_PATH_SYSTEM
        dlerror(); // clear
        
        // 在/odm/lib(64)/hw/，/vendor/lib(64)/hw/，/system/lib(64)/hw/
        // 寻找android.hardware.audio@2.0-impl.so并用dlopen打开，执行
        // HIDL_FETCH_IDevicesFactory函数,创建DevicesFactory实例
        for (const std::string &path : {
            HAL_LIBRARY_PATH_ODM, HAL_LIBRARY_PATH_VENDOR, HAL_LIBRARY_PATH_SYSTEM
        }) {
            std::vector<std::string> libs = search(path, prefix, ".so");
            for (const std::string &lib : libs) {
                const std::string fullPath = path + lib;
                // If sphal namespace is available, try to load from the
                // namespace first. If it fails, fall back to the original
                // dlopen, which loads from the current namespace.
                if (sphal_namespace != nullptr && path != HAL_LIBRARY_PATH_SYSTEM) {
                    const android_dlextinfo dlextinfo = {
                        .flags = ANDROID_DLEXT_USE_NAMESPACE,
                        // const_cast is dirty but required because
                        // library_namespace field is non-const.
                        .library_namespace = const_cast<android_namespace_t*>(sphal_namespace),
                    };
                    handle = android_dlopen_ext(fullPath.c_str(), dlMode, &dlextinfo);
                    if (handle == nullptr) {
                        const char* error = dlerror();
                        LOG(WARNING) << "Failed to dlopen " << lib << " from sphal namespace:"
                                     << (error == nullptr ? "unknown error" : error);
                    } else {
                        LOG(DEBUG) << lib << " loaded from sphal namespace.";
                    }
                }
                if (handle == nullptr) {
                    handle = dlopen(fullPath.c_str(), dlMode);
                }
                if (handle == nullptr) {
                    const char* error = dlerror();
                    LOG(ERROR) << "Failed to dlopen " << lib << ": "
                               << (error == nullptr ? "unknown error" : error);
                    continue;
                }
                IBase* (*generator)(const char* name);
                *(void **)(&generator) = dlsym(handle, sym.c_str());
                if(!generator) {
                    const char* error = dlerror();
                    LOG(ERROR) << "Passthrough lookup opened " << lib
                               << " but could not find symbol " << sym << ": "
                               << (error == nullptr ? "unknown error" : error);
                    dlclose(handle);
                    continue;
                }
                IBase *interface = (*generator)(name.c_str());
                if (interface == nullptr) {
                    dlclose(handle);
                    continue; // this module doesn't provide this instance name
                }
                registerReference(fqName, name);
                return interface;
            }
        }
        return nullptr;
    }
```
根据android.hardware.audio@2.0::IDevicesFactory字串拼接出android.hardware.audio@2.0-impl.so并找到打开，执行HIDL_FETCH_IDevicesFactory函数创建DevicesFactory返回。然后将DevicesFactory实例作为BsDevicesFactory的参数，将其封装返回，最后会执行IDevicesFactory的registerAsService。
```cpp
// out/soong/.intermediates/hardware/interfaces/audio/2.0/ \
// android.hardware.audio@2.0_genc++/gen/android/hardware/audio/2.0/DevicesFactoryAll.cpp
::android::status_t IDevicesFactory::registerAsService(const std::string &serviceName) {
    ::android::hardware::details::onRegistration(
                       "android.hardware.audio@2.0", "IDevicesFactory", serviceName);

    const ::android::sp<::android::hidl::manager::V1_0::IServiceManager> sm
            = ::android::hardware::defaultServiceManager();
    if (sm == nullptr) {
        return ::android::INVALID_OPERATION;
    }
    ::android::hardware::Return<bool> ret = sm->add(serviceName.c_str(), this);
    return ret.isOk() && ret ? ::android::OK : ::android::UNKNOWN_ERROR;
}
```
在registerAsService中先获取HwServiceManager的代理BpHwServiceManager，然后通过代理将IDevicesFactory注册到hwservicemanager。
```cpp
// system/libhidl/transport/ServiceManagerment.cpp
sp<IServiceManager> defaultServiceManager() {
    {
        AutoMutex _l(details::gDefaultServiceManagerLock);
        if (details::gDefaultServiceManager != NULL) {
            return details::gDefaultServiceManager;
        }

        if (access("/dev/hwbinder", F_OK|R_OK|W_OK) != 0) {
            // HwBinder not available on this device or not accessible to
            // this process.
            return nullptr;
        }

        waitForHwServiceManager();

        while (details::gDefaultServiceManager == NULL) {
            details::gDefaultServiceManager =
                    fromBinder<IServiceManager, BpHwServiceManager, BnHwServiceManager>(
                        ProcessState::self()->getContextObject(NULL));
            if (details::gDefaultServiceManager == NULL) {
                LOG(ERROR) << "Waited for hwservicemanager, but got nullptr.";
                sleep(1);
            }
        }
    }

    return details::gDefaultServiceManager;
}
```
我们看到关键代码是fromBinder函数，此函数会根据localBinder()的返回值创建代理或本地Stub服务。这里会创建代理BpHwServiceManager。

如下两个函数很关键，是将服务转化为Binder对象及将其还原。
```cpp
// system/libhidl/transport/include/hidl/HidlBinderSupport.h
// Construct a smallest possible binder from the given interface.
// If it is remote, then its remote() will be retrieved.
// Otherwise, the smallest possible BnChild is found where IChild is a subclass of IType
// and iface is of class IChild. BnChild will be used to wrapped the given iface.
// Return nullptr if iface is null or any failure.
template <typename IType, typename ProxyType>
sp<IBinder> toBinder(sp<IType> iface) {
    IType *ifacePtr = iface.get();
    if (ifacePtr == nullptr) {
        return nullptr;
    }
    if (ifacePtr->isRemote()) {
        return ::android::hardware::IInterface::asBinder(static_cast<ProxyType *>(ifacePtr));
    } else {
        std::string myDescriptor = details::getDescriptor(ifacePtr);
        if (myDescriptor.empty()) {
            // interfaceDescriptor fails
            return nullptr;
        }
        auto func = details::gBnConstructorMap.get(myDescriptor, nullptr);
        if (!func) {
            return nullptr;
        }
        return sp<IBinder>(func(static_cast<void *>(ifacePtr)));
    }
}

template <typename IType, typename ProxyType, typename StubType>
sp<IType> fromBinder(const sp<IBinder>& binderIface) {
    using ::android::hidl::base::V1_0::IBase;
    using ::android::hidl::base::V1_0::BnHwBase;

    if (binderIface.get() == nullptr) {
        return nullptr;
    }
    if (binderIface->localBinder() == nullptr) {
        return new ProxyType(binderIface);
    }
    sp<IBase> base = static_cast<BnHwBase*>(binderIface.get())->getImpl();
    if (details::canCastInterface(base.get(), IType::descriptor)) {
        StubType* stub = static_cast<StubType*>(binderIface.get());
        return stub->getImpl();
    } else {
        return nullptr;
    }
}
```
接下来执行BpHwServiceManager的add函数。其中关键代码toBinder函数，BsDevicesFactory继承自IDevicesFactory，IDevicesFactory的isRemote()会返回false，所以会将传入的BsDevicesFactory服务封装成BnHwDevicesFactory服务，然后通过Parcel类型_hidl_data传递，封装完成则通过remote()->transact进入Binder驱动，通过Binder驱动传递到hwservicemanager。
```cpp
// out/soong/.intermediates/system/libhidl/transport/manager/1.0/ \
// android.hidl.manager@1.0_genc++/gen/android/hidl/manager/1.0/ServiceManagerAll.cpp
::android::hardware::Return<bool> BpHwServiceManager::add(
           const ::android::hardware::hidl_string& name, 
           const ::android::sp<::android::hidl::base::V1_0::IBase>& service) {
    atrace_begin(ATRACE_TAG_HAL, "HIDL::IServiceManager::add::client");

    ......

    ::android::hardware::Parcel _hidl_data;
    ::android::hardware::Parcel _hidl_reply;
    ::android::status_t _hidl_err;
    ::android::hardware::Status _hidl_status;

    bool _hidl_out_success;

    _hidl_err = _hidl_data.writeInterfaceToken(IServiceManager::descriptor);
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

    if (service == nullptr) {
        _hidl_err = _hidl_data.writeStrongBinder(nullptr);
    } else {
        ::android::sp<::android::hardware::IBinder> _hidl_binder = 
                ::android::hardware::toBinder<
                ::android::hidl::base::V1_0::IBase, 
                ::android::hidl::base::V1_0::BpHwBase>(service);
        if (_hidl_binder.get() != nullptr) {
            _hidl_err = _hidl_data.writeStrongBinder(_hidl_binder);
        } else {
            _hidl_err = ::android::UNKNOWN_ERROR;
        }
    }
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    ::android::hardware::ProcessState::self()->startThreadPool();
    _hidl_err = remote()->transact(2 /* add */, _hidl_data, &_hidl_reply);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    _hidl_err = ::android::hardware::readFromParcel(&_hidl_status, _hidl_reply);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    if (!_hidl_status.isOk()) { return _hidl_status; }

    _hidl_err = _hidl_reply.readBool(&_hidl_out_success);
    if (_hidl_err != ::android::OK) { goto _hidl_error; }

    atrace_end(ATRACE_TAG_HAL);
    
    ......

    _hidl_status.setFromStatusT(_hidl_err);
    return ::android::hardware::Return<bool>(_hidl_out_success);

_hidl_error:
    _hidl_status.setFromStatusT(_hidl_err);
    return ::android::hardware::Return<bool>(_hidl_status);
}
```
经过Binder驱动的传递，返回用户空间会执行BnHwServiceManager的onTransact函数，首先执行fromBinder函数，将BsDevicesFactory还原，因为这里_hidl_data存放的是BnHwDevicesFactory的，BnHwDevicesFactory间接继承自BHwBinder，所以localBinder()返回不为空，即还原BsDevicesFactory，最后将其保存在ServiceManager(system/hwservicemanager)的std::map<std::string, PackageInterfaceMap> mServiceMap中。
```cpp
// out/soong/.intermediates/system/libhidl/transport/manager/1.0/ \
// android.hidl.manager@1.0_genc++/gen/android/hidl/manager/1.0/ServiceManagerAll.cpp
::android::status_t BnHwServiceManager::onTransact(
        uint32_t _hidl_code,
        const ::android::hardware::Parcel &_hidl_data,
        ::android::hardware::Parcel *_hidl_reply,
        uint32_t _hidl_flags,
        TransactCallback _hidl_cb) {
    ::android::status_t _hidl_err = ::android::OK;

    switch (_hidl_code) {
        ......
        case 2 /* add */:
        {
            if (!_hidl_data.enforceInterface(IServiceManager::descriptor)) {
                _hidl_err = ::android::BAD_TYPE;
                break;
            }

            const ::android::hardware::hidl_string* name;
            ::android::sp<::android::hidl::base::V1_0::IBase> service;

            size_t _hidl_name_parent;

            _hidl_err = _hidl_data.readBuffer(
                               sizeof(*name), &_hidl_name_parent,
                               reinterpret_cast<const void **>(&name));

            if (_hidl_err != ::android::OK) { break; }

            _hidl_err = ::android::hardware::readEmbeddedFromParcel(
                    const_cast<::android::hardware::hidl_string &>(*name),
                    _hidl_data,
                    _hidl_name_parent,
                    0 /* parentOffset */);

            if (_hidl_err != ::android::OK) { break; }

            {
                ::android::sp<::android::hardware::IBinder> _hidl_service_binder;
                _hidl_err = _hidl_data.readNullableStrongBinder(&_hidl_service_binder);
                if (_hidl_err != ::android::OK) { break; }

                service = ::android::hardware::fromBinder<
                       ::android::hidl::base::V1_0::IBase,
                       ::android::hidl::base::V1_0::BpHwBase,
                       ::android::hidl::base::V1_0::BnHwBase>(_hidl_service_binder);
            }

            atrace_begin(ATRACE_TAG_HAL, "HIDL::IServiceManager::add::server");
            ......

            bool _hidl_out_success = _hidl_mImpl->add(*name, service);

            ::android::hardware::writeToParcel(::android::hardware::Status::ok(), _hidl_reply);

            _hidl_err = _hidl_reply->writeBool(_hidl_out_success);
            /* _hidl_err ignored! */

            atrace_end(ATRACE_TAG_HAL);
            ......

            _hidl_cb(*_hidl_reply);
            break;
        }
        ......
    }

    ......
}
```
从上面流程可以看出，这和之前的Binder通信很类似，所以只要之前学好了Binder的原理，就能快速了解native和hal的通信原理及流程。
## 加载HAL so
回想在AudioPolicyService启动的时候，会mDevicesFactoryHal->openDevice(name, &dev),其实最终会调用DevicesFactory的loadAudioInterface，即打开audio.primary.default.so等lib。
```cpp
// hardware/interface/audio/2.0/default/DevicesFactory.cpp
int DevicesFactory::loadAudioInterface(const char *if_name, audio_hw_device_t **dev)
{
    const hw_module_t *mod;
    int rc;
    rc = hw_get_module_by_class(AUDIO_HARDWARE_MODULE_ID, if_name, &mod);
    if (rc) {
        ALOGE("%s couldn't load audio hw module %s.%s (%s)", __func__,
                AUDIO_HARDWARE_MODULE_ID, if_name, strerror(-rc));
        goto out;
    }
    rc = audio_hw_device_open(mod, dev);
    if (rc) {
        ALOGE("%s couldn't open audio hw device in %s.%s (%s)", __func__,
                AUDIO_HARDWARE_MODULE_ID, if_name, strerror(-rc));
        goto out;
    }
    if ((*dev)->common.version < AUDIO_DEVICE_API_VERSION_MIN) {
        ALOGE("%s wrong audio hw device version %04x", __func__, (*dev)->common.version);
        rc = -EINVAL;
        audio_hw_device_close(*dev);
        goto out;
    }
    return OK;
out:
    *dev = NULL;
    return rc;
}
// Methods from ::android::hardware::audio::V2_0::IDevicesFactory follow.
Return<void> DevicesFactory::openDevice(IDevicesFactory::Device device, 
                                        openDevice_cb _hidl_cb)  {
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