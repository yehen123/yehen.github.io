title: next.js配置测试环境的解决方案
tags: [next.js]
categories: []
date: 2020-11-30 16:06:00
---
一般来说，next框架在不同环境中使用**.env.local**中的不同配置即可区分，但是运维同学表示并不想多写这一行代码，只能自己动手啦。

>  项目中的代码需要在3个环境中运行：开发环境、测试环境、生产环境，而next中通过`process.env.NODE_ENV`的方式只支持development和production两种方式，那么如何解决测试环境的问题呢？

## 开发环境

# server/server.js
```js
const express = require('express')
const next = require('next')
const { createProxyMiddleware } = require("http-proxy-middleware")

const port = process.env.PORT || 3000
const dev = process.env.NODE_ENV !== 'production'
const app = next({ dev })
const handle = app.getRequestHandler()

const apiPaths = {
  '/api': {
    target: 'http://www.target.com',
    pathRewrite: {
      '^/api': '/'
    },
    changeOrigin: true
  }
}

const isDevelopment = process.env.NODE_ENV !== 'production'

app.prepare().then(() => {
  const server = express()
 
  if (isDevelopment && apiPaths) {
    Object.keys(apiPaths).forEach(function(context) {
      server.use(createProxyMiddleware(context, apiPaths[context]))
    })
  }

  server.all('*', (req, res) => {
    return handle(req, res)
  })

  server.listen(port, (err) => {
    if (err) throw err
    console.log(`> Ready on http://localhost:${port}`)
  })
}).catch(err => {
    console.log('Error:::::', err)
})
```
# package.json
```
"scripts": {
  "dev": "cross-env NODE_ENV=development node server/server.js",
  ...
},
```

```process.env.NODE_ENV```为```development```时为请求url添加**/api**前缀即可

## 测试环境

测试环境与生产环境同使用`next start`进行构建，`cross-env NODE_ENV=staging next start,`是无效的，next.js在执行start时会将其覆盖，因为区分测试环境和生产环境则是个棘手的问题。

生产环境和测试环境部署在不同的域名下，解决方案可以通过当前域名来作为区分，而node环境是没有window对象和域名的，因此只要解决代码在node环境下运行时的问题即可解决。

# swr全局配置

_app.js

```jsx
import fetcher from 'utils/fetcher'

<SWRConfig 
  value={{
    fetcher,
  }}
>
  <Component {...pageProps} />
</SWRConfig>
```

# 封装fetcher

>swr会在每次请求时调用fetcher，因此通过闭包将当前环境下的host持久化储存即可

```js
const fetcher = () => {
  let host = ''
  try {
    // node环境同样会被执行，因为没有window对象在此会抛出一个error
    if(window) {
      if(process.env.NODE_ENV === 'development') {
        // 开发环境
        host = '/api'
      } else {
        // 通过当前域名来区分不同环境
        host = window.location.host.includes('test') ? 'http://www.test.com' : 'https://www.prod.com'
      }
    }
  } catch(e) {}

  return (api, ...args) => fetch(host+api, ...args).then(res => res.json())
}

// 闭包持久化储存host，避免每次调用时执行
export default fetcher()
```

---
end