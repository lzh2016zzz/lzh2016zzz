---
title: React入门之request
tags: 
   - 拾遗
   - react
---

## 准备工作


安装umi-request 

umi-request是一个上手简单,可拓展性强的request库,这个库可以向服务器发起`get`,`post`,`delete`,`put`请求,或者把数据推送到服务器上
```bash
yarn add umi-request
#或者
npm install --save umi-request
```

## 编写request.ts

```typescript
/**
 * @fileoverview 请求封装
 * @description 错误码, 拦截器等处理
 * @author laizeh
 * @date 2021/01/15
 */
import { extend } from 'umi-request';
import { notification } from 'antd';

const isDev = process.env.NODE_ENV === 'development'

//这里定义了http code对应的message
const codeMessage: {
  [propsName: string]: string;
} = {
  200: '服务器成功返回请求的数据。',
  201: '新建或修改数据成功。',
  202: '一个请求已经进入后台排队（异步任务）。',
  204: '删除数据成功。',
  400: '发出的请求有错误，服务器没有进行新建或修改数据的操作。',
  401: '用户没有权限（令牌、用户名、密码错误）。',
  403: '用户得到授权，但是访问是被禁止的。',
  404: '发出的请求针对的是不存在的记录，服务器没有进行操作。',
  406: '请求的格式不可得。',
  410: '请求的资源被永久删除，且不会再得到的。',
  422: '当创建一个对象时，发生一个验证错误。',
  500: '服务器发生错误，请检查服务器。',
  502: '网关错误。',
  503: '服务不可用，服务器暂时过载或维护。',
  504: '网关超时。',
};

/**
 * 统一异常处理程序
 */
const errorHandler = (error: any) => {
  const { response } = error;

  if (response && response.status) {
    const errorText = codeMessage[response.status] || response.statusText;
    const { status, url } = response;
    notification.error({
      message: `请求错误 ${status}: ${url}`,
      description: errorText,
    });
  }


  return response;
};
const request = extend({
  prefix: isDev ? "" : "http://www.laizeh.com:8080",
  errorHandler,
  // 默认错误处理
  credentials: 'include', // 默认请求是否带上cookie
});

export default request;

```


## 使用

#### 定义请求接口

```typescript
export async function initIndexApi(page:{pageNo :number,pageSize :number}) {
  return <Promise<ApiResult<Page<ListArticle>>>>request('/api/blog/initIndex', {
    method: 'post',
    data: page,
  });
}
```
这里定义了一个方法,这个方法向`/api/blog/initIndex`发起`post`请求,返回一个`Promise`(`Promise`的`result`由服务端返回的数据结构决定)

#### 调用
在`useEffect()钩子中使用`
```typescript
useEffect(() => {
    setLoad(true);
    initIndexApi(filter)
      .then((data) => {
           //这里处理业务逻辑
        }
      })
      .finally(() => {
          //这里的代码会在调用方法之后被执行,无论调用方法成功或者失败
          setLoad(false);
      });
  }, [filter]);
```


