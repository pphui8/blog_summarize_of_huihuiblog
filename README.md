## 编写 huihuiblog 的心得体会
#### huihuiblog 是啥
huihuiblog 是一个基于 GitHub API 的前后端分离项目。大致的搭建思路是 `服务器获取目录 -> GitHub 获取博客内容。` [blog地址](https://blog.pphui8.me)

##### [前端项目地址](https://github.com/pphui8/huihuiblog)
前端：React

##### [后端项目地址](https://github.com/pphui8/phidippides)
后端：Rust + rocket  
WebServer：Nginx  

### 踩到的坑
##### 1. 新版本的 `rocket`  
新版的 rocket 并没有更新说明文档。故回退到 `rocket=“20.0.0”` 。此版本的 `rocket` 也不需要开启 `nightly` 模式。  

##### 2. `Nginx` 的文件路由与 `React` 路由冲突

> 导致的问题：刷新路由界面显示 `404`  

更改了路由传参的方法，使用 `State` 进行路由间的参数传递而不是显式的传递参数。

##### 3. `React` 自带的路由缓存导致组件没有及时更新

> 导致的问题： 进入其它的博文页面仍显示之前的内容 

应该要使用 `diff` 算法来对不同的博文显示不同的 `id` 值，但是设置在 `Markdown` (虚拟元素) 中似乎不起效果 (也可能是算法没写对)。最后突然想到在页面加载前先把存储元素的变量设置为空，然后进行数据获取就能正常渲染力 (天才すぎ、オレ)

##### 4. 通过 `Nginx` 处理的允许请求跨域
最先使用了 `React-middle-ware` 从前端进行了跨域设置，后来才知道这玩意只能实现生产环境的请求跨域。`Rocket` 的跨域设置属实是有点复杂 (还要开启 `Nightly`)。最后使用 `Nginx` 实现了跨域。然而网上的教程大部分没有写全，而且一篇文章到处抄，这里稍微讲一下下怎么使用 `Nginx` 进行的跨域。

> 其实在前端 `fetch` 请求设置为 `no-cors` 也能实现前端跨域并能带到部署环境中。但是此方式请求的请求体仅支持传输 `text` 数据而不支持 `json`。

1. 首先是 https 的基础设置
```json
server {
    listen       443 ssl;
    server_name  api.pphui8.me;

    ssl_certificate "xxx";
    ssl_certificate_key "xxx";

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;
    ...
```

2. 在文件路由中设置允许跨域
```json
location / {
    proxy_pass  http://127.0.0.1:xxxx;

    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
    add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';

    if ($request_method = 'OPTIONS') {
        return 204;
    }
}
```

3. 如果需要带路由的跨域（或者是多个跨域设置）
```json
location /addcomment {
        proxy_pass  http://127.0.0.1:8000;

        add_header Access-Control-Allow-Origin *;
        add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
        add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';

        if ($request_method = 'OPTIONS') {
            return 204;
        }
    }
```

##### 5. `Rsut` 中 `mysql` 数据连接池的处理
最开始是直接使用一次请求创建一次连接池 (反正够快来着..)。由于 `Rust` 并不能随意设置全局变量，而静态全局变量不可使用非静态函数进行初始化。最后使用 `Lazy` 库实现了懒加载。
```Rust
lazy_static! {
    static ref SQLPOOL: mysql::Pool = Pool::new(BLOG_URL).unwrap();
}
```