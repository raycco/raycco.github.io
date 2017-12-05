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
在如上的启动代码中我们看到，HAL服务端调用registerPassthroughServiceImplementation实现注册，这函数的实现在system/libhidl/transport/include/hidl/LegacySupport.h,这是一个模板方法，这里传入IDevicesFactory接口。
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
const char* IDevicesFactory::descriptor("android.hardware.audio@2.0::IDevicesFactory");

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
根据android.hardware.audio@2.0::IDevicesFactory字串拼接出android.hardware.audio@2.0-impl.so并找到打开，执行HIDL_FETCH_IDevicesFactory函数创建DevicesFactory返回。得到BsDevicesFactory实例后，会执行其registerAsService。
```cpp
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
获取HwServiceManager将自己注册到hwservicemanager。这里涉及binder通信机制，关于binder通信，是另外一个比较大的topic，在这里一两句说不清楚，所以后续有时间再专门记录。
## 加载HAL so
回想在AudioPolicyService启动的时候，会mDevicesFactoryHal->openDevice(name, &dev),其实最终会调用DevicesFactory的loadAudioInterface，即打开audio.primary.default.so等lib。
```cpp
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