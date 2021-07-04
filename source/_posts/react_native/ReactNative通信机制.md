---
title: ReactNative-Java与JavaScript之间的通信
date: 2018-12-09 09:30:33
layout: config.default_layout
categories: 'React Native'
tags:
- React Native
---
# ReactNative-Java与JavaScript之间的通信

## 几个重要的类



- Java层 ReactContext(ReactApplicationContext)： React Native 封装后的 Android Context，通过其访问设置 RN 包装起来的核心类实现等；

- Java层 ReactInstanceManager(ReactInstanceManagerImpl)： RN对 Android 层暴露的大内总管，负责掌管 CatalystInstanceImpl 实例、ReactRootView、Activity 生命周期等；

- Java/C++层 CatalystInstance(CatalystInstanceImpl)： RN Java、C++、JS通信总舵主，统管 JS、Java 核心 Module 映射表、回调等等，三端入口与桥梁；

- C++层 NativeToJsBridge： Java 调用 JS 的桥梁，用来调用 JS Module、回调 Java（通过JsToNativeBridge）等；

- C++层 JsToNativeBridge： JS 调用 Java 的桥梁，用来调用 Java Module等；

- C++层 JSCExecutor： 掌管 Webkit 的 JavaScriptCore，JS 与 C++ 的转换桥接都在这里中转处理；

- JS层 MessageQueue： 队列栈，用来处理 JS 的调用队列、调用 Java 或者 JS Module 的方法、处理回调、管理 JS Module 等；

- 多层 JavaScriptModule/BaseJavaModule(NativeModule)： 双端字典映射表中的模块，负责 Java/JS 到彼此的映射调用格式申明，由 CatalystInstance 统管；



## Java调用JavaScript

![img](https://cdn.nlark.com/yuque/0/2018/png/199077/1543751641173-8a70a3ba-ff6e-41af-bbfc-3bc04fc848d2.png)

```c++

void JSCExecutor::callFunction(const std::string& moduleId, const std::string& methodId, const folly::dynamic& arguments) {
  ......
  auto result = [&] {
    try {
        //m_callFunctionReturnFlushedQueueJS来自于JSCExecutor::bindBridge()方法中初始化，JSCExecutor::bindBridge()是在前面分析启动流程时被调用的，实质是负责通过 Webkit JSC 拿到 JS 端代码的相关对象和方法引用，譬如拿到 JS 端 BatchedBridge.js 的 __fbBatchedBridge 属性与 MessageQueue.js 的 callFunctionReturnFlushedQueue 方法引用。此处实质为调用 MessageQueue.js 的 callFunctionReturnFlushedQueue 方法。
      return m_callFunctionReturnFlushedQueueJS->callAsFunction({
        Value(m_context, String::createExpectingAscii(moduleId)),
        Value(m_context, String::createExpectingAscii(methodId)),
        Value::fromDynamic(m_context, std::move(arguments))
      });
    } catch (...) {
      std::throw_with_nested(
        std::runtime_error("Error calling function: " + moduleId + ":" + methodId));
    }
  }();
    //调用 native 模块，暂时忽略，下一小节解释，这里重点关注 Java 调用 JS 通信
  callNativeModules(std::move(result));
}
```



## JavaScript调用Java



### ![img](https://cdn.nlark.com/yuque/0/2018/png/199077/1543751615410-914290d1-f1a8-44e0-89e3-26176a5fe11a.png)Sample



```js
//通过NativeModules拿到ToastAndroid
import { NativeModules } from 'react-native';
module.exports = NativeModules.ToastAndroid;

//使用的地方在JS中相关逻辑处调用，官方文档标准
import ToastAndroid from './ToastAndroid';
ToastAndroid.show('Awesome', ToastAndroid.SHORT);
```



![img](file:///var/folders/v0/8dgsh1jd74qfkddq5pcp_znc0000gn/T/abnerworks.Typora/image-20181202165046268.png?lastModify=1543751522)

```java
function genModule(config: ?ModuleConfig, moduleID: number): ?{name: string, module?: Object} {
  ......
  //通过JSC拿到C++中从Java端获取的Java的Module映射表包装配置类
  const [moduleName, constants, methods, promiseMethods, syncMethods] = config;
  ......
  const module = {};
  //遍历构建module的属性方法
  methods && methods.forEach((methodName, methodID) => {
    const isPromise = promiseMethods && arrayContains(promiseMethods, methodID);
    const isSync = syncMethods && arrayContains(syncMethods, methodID);
    invariant(!isPromise || !isSync, 'Cannot have a method that is both async and a sync hook');
    const methodType = isPromise ? 'promise' : isSync ? 'sync' : 'async';
    //生成Module的函数方法
    module[methodName] = genMethod(moduleID, methodID, methodType);
  });
  Object.assign(module, constants);
  ......
  //返回一个
  return { name: moduleName, module };
}

enqueueNativeCall(moduleID: number, methodID: number, params: Array<any>, onFail: ?Function, onSucc: ?Function) {
    ......
    this._callID++;
    //_queue是个队列，用于存放调用的模块、方法、参数信息
    //把JS准备调用Java的模块名、方法名、调用参数放到数组里存起来
    this._queue[MODULE_IDS].push(moduleID);
    this._queue[METHOD_IDS].push(methodID);
    ......
    this._queue[PARAMS].push(params);

    const now = new Date().getTime();
    //如果5ms内有多个方法调用就先待在队列里防止过高频率，否则调用C++的nativeFlushQueueImmediate方法
    if (global.nativeFlushQueueImmediate &&
        now - this._lastFlush >= MIN_TIME_BETWEEN_FLUSHES_MS) {
      global.nativeFlushQueueImmediate(this._queue);
      this._queue = [[], [], [], this._callID];
      this._lastFlush = now;
    }
    ......
}
```