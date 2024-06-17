---
description: 对外接口基于Node.js N-API规范开发的原生模块扩展开发框架。
---

# 07-NAPI接口

## 简介

NAPI 的概念源自 Nodejs，为了实现 javascript 脚本与 C++ 库之间的相互调用，Nodejs 对 V8 引擎的 api 做了一层封装，称为 NAPI。

&#x20;Nodejs 官网（[https://nodejs.org/dist/latest-v20.x/docs/api/n-api.html](https://link.zhihu.com/?target=https%3A//nodejs.org/dist/latest-v20.x/docs/api/n-api.html)）上查看各种 NAPI 接口定义说明。

<figure><img src=".gitbook/assets/image (22).png" alt=""><figcaption></figcaption></figure>

NAPI 接口本身是 C++ 语言实现的，这些接口可以帮助 C++ 代码创建 JS 变量，或访问 JavaScript 运行环境中的 JS 变量与方法。

但是OpenHarmony的napi并非是nodejs的napi，OpenHarmony 系统沿用了 NAPI 的接口定义形式，但每个接口的内部实现都进行了重写。

### NAPI库的简单实现 <a href="#h_680968108_4" id="h_680968108_4"></a>

### c++实现

```cpp
#include "napi/native_api.h"

//接口实现
static napi_value Add(napi_env env, napi_callback_info info)
{
    size_t requireArgc = 2;
    size_t argc = 2;
    napi_value args[2] = {nullptr};

    napi_get_cb_info(env, info, &argc, args , nullptr, nullptr);
    //参数解析
    napi_valuetype valuetype0;
    napi_typeof(env, args[0], &valuetype0);
    napi_valuetype valuetype1;
    napi_typeof(env, args[1], &valuetype1);
    //参数获取值
    double value0;
    napi_get_value_double(env, args[0], &value0);
    double value1;
    napi_get_value_double(env, args[1], &value1);
    //逻辑实现
    napi_value sum;
    napi_create_double(env, value0 + value1, &sum);
    return sum;
}

//初始化
EXTERN_C_START
static napi_value Init(napi_env env, napi_value exports)
{
    napi_property_descriptor desc[] = {
        { "add", nullptr, Add, nullptr, nullptr, nullptr, napi_default, nullptr }
    };
    napi_define_properties(env, exports, sizeof(desc) / sizeof(desc[0]), desc);
    return exports;
}
EXTERN_C_END

//module定义
static napi_module demoModule = {
    .nm_version = 1,
    .nm_flags = 0,
    .nm_filename = nullptr,
    .nm_register_func = Init,
    .nm_modname = "entry",
    .nm_priv = ((void*)0),
    .reserved = { 0 },
};

//module注册
extern "C" __attribute__((constructor)) void RegisterEntryModule(void)
{
    napi_module_register(&demoModule);
}

```

### ts接口定义

```javascript
export const add: (a: number, b: number) => number;
```

### 接口使用

```javascript
import testNapi from 'libentry.so';

 hilog.info(0x0000, 'testTag', 'Test NAPI 2 + 3 = %{public}d', testNapi.add(2, 3));
```

### 编译后结构 <a href="#h_680968108_4" id="h_680968108_4"></a>

<figure><img src=".gitbook/assets/image (23).png" alt=""><figcaption></figcaption></figure>

```cmake
# the minimum version of CMake.
cmake_minimum_required(VERSION 3.4.1)
project(MyApplication6)

set(NATIVERENDER_ROOT_PATH ${CMAKE_CURRENT_SOURCE_DIR})

include_directories(${NATIVERENDER_ROOT_PATH}
                    ${NATIVERENDER_ROOT_PATH}/include)

add_library(entry SHARED hello.cpp)
//链接libace_napi实现
target_link_libraries(entry PUBLIC libace_napi.z.so)
```

### NAPI 库的导入原理 <a href="#h_680968108_4" id="h_680968108_4"></a>



### 整体流程

<figure><img src=".gitbook/assets/image (25).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (27).png" alt=""><figcaption></figcaption></figure>

其中libentry.so如何生效的呢？就是通过 dlopen () 将对应的 接口实现.z.so 加载到应用进程中，foundation/arkui/napi/native\_engine/impl/ark中ark\_native\_engine.cpp

<figure><img src=".gitbook/assets/image (24).png" alt=""><figcaption></figcaption></figure>

每个应用进程的 JS 引擎中，都注册了一个 “requireNapi” 函数，当应用调用此方法时，JS 引擎就会通过 NAPI 框架的 moduleManager 类去处理 so 库的加载。moduleManager 内部最终是找到了 /system/lib/module 下对应的 so 文件，并通过 dlopen () 的方式加载到应用进程中。

而index.ets 被编译成 index.js 后，import 关键字也被转为了 “requireNapi”，这样 JS 引擎在执行这行代码时，就会去调用注册的导入处理函数了。

<figure><img src=".gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>

当调用 **dlopen()** 方法加载 `libentry.so` 时，会先调用 **RegisterEntryModule()** 方法，在该方法内部调用了 **napi\_module\_register()**

```cpp

NAPI_EXTERN void napi_module_register(napi_module* mod)
{
    if (mod == nullptr) {
        HILOG_ERROR("mod is nullptr");
        return;
    }

    NativeModuleManager* moduleManager = NativeModuleManager::GetInstance();
    NativeModule module;

    module.version = mod->nm_version;
    module.fileName = mod->nm_filename;
    module.name = mod->nm_modname;
    module.registerCallback = (RegisterCallback)mod->nm_register_func;

    moduleManager->Register(&module);
}

```

注册

```cpp

void NativeModuleManager::Register(NativeModule* nativeModule)
{
    if (nativeModule == nullptr) {
        HILOG_ERROR("nativeModule value is null");
        return;
    }

    HILOG_DEBUG("native module name is '%{public}s'", nativeModule->name);
    std::lock_guard<std::mutex> lock(nativeModuleListMutex_);
    if (!CreateNewNativeModule()) {
        HILOG_ERROR("create new nativeModule failed");
        return;
    }

    const char *nativeModuleName = nativeModule->name == nullptr ? "" : nativeModule->name;
    std::string appName = prefix_ + "/" + nativeModuleName;
    const char *tmpName = isAppModule_ ? appName.c_str() : nativeModuleName;
    char *moduleName = strdup(tmpName);
    if (moduleName == nullptr) {
        HILOG_ERROR("strdup failed. tmpName is %{public}s", tmpName);
        return;
    }

    lastNativeModule_->version = nativeModule->version;
    lastNativeModule_->fileName = nativeModule->fileName;
    lastNativeModule_->isAppModule = isAppModule_;
    lastNativeModule_->name = moduleName;
    lastNativeModule_->refCount = nativeModule->refCount;
    lastNativeModule_->registerCallback = nativeModule->registerCallback;
    lastNativeModule_->getJSCode = nativeModule->getJSCode;
    lastNativeModule_->getABCCode = nativeModule->getABCCode;
    lastNativeModule_->next = nullptr;
    lastNativeModule_->moduleLoaded = true;

    HILOG_DEBUG("register module name is '%{public}s', isAppModule is %{public}d",
        lastNativeModule_->name, isAppModule_);
}

```

&#x20;**Register()** 方法负责把传递进来的 `NativeModule` 加入链表的末尾。

其中在 **napi\_module\_register()** 方法内部把 `mod` 中配置的 `nm_register_func` 强制转换成 `RegisterCallback` 后赋值给了 `NativeModule` 的 `registerCallback`

`而`**nm\_register\_func** 就是在 `hello.cpp` 中配置的 **Init()** 方法

<figure><img src=".gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>

这里会再调用napi\_define\_properties（）方法

```cpp
NAPI_EXTERN napi_status napi_define_properties(napi_env env,
                                               napi_value object,
                                               size_t property_count,
                                               const napi_property_descriptor* properties)
{
    NAPI_PREAMBLE(env);
    CHECK_ARG(env, object);
    CHECK_ARG(env, properties);

    auto nativeValue = LocalValueFromJsValue(object);
    RETURN_STATUS_IF_FALSE(env, nativeValue->IsObject() || nativeValue->IsFunction(),
        napi_object_expected);
    auto vm = reinterpret_cast<NativeEngine*>(env)->GetEcmaVm();
    Local<panda::ObjectRef> nativeObject = nativeValue->ToObject(vm);

    auto nativeProperties = reinterpret_cast<const NapiPropertyDescriptor*>(properties);
    for (size_t i = 0; i < property_count; i++) {
        if (nativeProperties[i].utf8name == nullptr) {
            auto name = LocalValueFromJsValue(nativeProperties[i].name);
            RETURN_STATUS_IF_FALSE(env, !name.IsEmpty() && (name->IsString() || name->IsSymbol()), napi_name_expected);
        }
        NapiDefineProperty(env, nativeObject, nativeProperties[i]);
    }
    return GET_RETURN_STATUS(env);
}
```

**napi\_define\_properties()** 方法内部循环遍历传递进来的每一个 `napi_property_descriptor`，把每一个 `napi_property_descriptor` 转化成 `NapiPropertyDescriptor` 的 `property` 并调用 **NapiDefineProperty()** 方法完成 JS 方法和 C++方法的映射。
