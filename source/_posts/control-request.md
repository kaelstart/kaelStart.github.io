---
title: 控制业务中重复请求
date: 2021-01-04 17:43:56
tags:
---

最近老大给了一个新的需求,需求是防止项目中表单有重复提交.当时拿到这个需求的时候,心里就想在全局请求封装的axios中增加一个全屏幕loading应该就可以了,但是当我把这个效果坐上去之后,发现整个项目都变得非常的难看,因为页面一直在loading中...
于是乎我想了另外一种办法,就是在封装的axios的拦截器中做一些处理.当发起请求时,请求还没有响应的时候,再次发起请求就会拦截或者说是取消掉这次的请求.

### 思路
    我们必须有一个请求池来保存每一次请求的记录,这样当我们在请求时,请求完成和请求失败的时候都能够做对应的处理.
    请求的时候需要往这个请求池中添加当前请求的唯一标识
    请求成功和请求失败的时候需要往这个请求池中移除当前请求的唯一标识

第一个问题是,我们应该如何如果把一个请求作为一个标识储存在请求池中间呢?其实也很简单,每个请求都会有url,method,params,data等参数.如果我们把一个请求的以上参数序列化之后作为一个标识放入请求池中,就可以解决请求池中每个请求的唯一性问题.
    
第二个问题是,如果在axios中的请求拦截器监听到了重复的请求,应该如何处理?我们去axios的文档中可以看到,axios有一个CancelToken的方法,当我们监听到了重复的请求的时候就往当前请求的配置中增加一个CancelToken,把这次请求的取消掉.
### axios的源码
``` bash
      lib/adapters/xhr
      if (config.cancelToken) {
      // Handle cancellation
      config.cancelToken.promise.then(function onCanceled(cancel) {
        if (!request) {
          return;
        }

        request.abort();
        reject(cancel);
        // Clean up request
        request = null;
      });
    }
```


### 请求池代码如下

``` bash
const ControlRequest = {
  requestRecord: {},
  resolver: {},
  rejecter: {},
  add(request) {
    const serialized = this.handleSerialize(request);
    this.requestRecord[serialized] = this.resolver[serialized] = this.rejecter[
      serialized
    ] = true;
  },
  get(request) {
    const serialized = this.handleSerialize(request);
    return this.requestRecord[serialized];
  },
  success(request) {
    const serialized = this.handleSerialize(request);
    this.resolver[serialized] && this.handleClean(request);
  },
  fail(request) {
    const serialized = this.handleSerialize(request);
    this.rejecter[serialized] && this.handleClean(request);
  },
  handleClean(request) {
    const serialized = this.handleSerialize(request);
    delete this.resolver[serialized];
    delete this.rejecter[serialized];
    delete this.requestRecord[serialized];
  },
  // 序列化每一个请求的标识
  handleSerialize(request) {
    const { method, params, body, url, data } = request;
    const sData = typeof data === 'string' ? JSON.parse(data || '{}') : data;
    return JSON.stringify({ method, params, body, url, sData });
  },
}
```


### 在业务中使用

``` bash
import axios from 'axios';
const CancelToken = axios.CancelToken;
const source = CancelToken.source();

// 请求拦截
instance.interceptors.request.use(
  function (config) {
    const requesting = ControlRequest.get(config);
    if (requesting) {
      config.cancelToken = source.token;
      source.cancel(config);
      return config;
    }

    ControlRequest.add(request);

    return request;
  },
  function (error) {
    return Promise.reject(error);
  },
);

// 响应拦截
instance.interceptors.response.use(
  function (response) {
    ControlRequest.success(response.config);
    return response;
  },
  function (error) {
    // 这里注意,如果是被axios取消的请求也会走到这个error中,我们要排除那些被axios取消的请求.
    !error.__CANCEL__ && ControlRequest.fail(error.config);

    return Promise.reject(error);
  },
);

```
初次写文章,可能还有很多地方没有考虑全.如果哪里写的不对,还请各位指点.

