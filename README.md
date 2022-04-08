# kong-plugin-demo

一个简单的 Kong 自定义插件示例，插件执行的逻辑是，读取 `X-Request-Base64` 请求 header 的字符串参数，然后对这个字符串执行 base64 解码，解码结果通过 header `X-Response-Decoded` 返回。

在腾讯云云原生网关Kong中安装该插件:

``` bash
git clone https://github.com/hongliang5316/kong-plugin-demo.git
cd kong-plugin-demo
# 将插件打成zip包
zip -q -r demo.zip demo
ls demo.zip
```

`demo.zip`可直接在腾讯云云原生网关Kong上进行自定义插件的上传以及安装

插件的配置可在`Konga`控制台中配置, 或者通过`Kong admin-api`来配置

以下是通过`Kong admin-api`的方式配置:

```
$ curl -s http://localhost:8001/plugins/enabled | grep demo
```

上面让 Kong 加载到这个插件，但还没有开启，没有真正执行。现在来试下开启这个插件。

通过调用 Kong 提供的 Admin API，让 Kong 创建了名为 `example.service` 的 `service`，`service` 指向的上游服务是 `http://httpbin.org`。然后，在 `example.service` 上添加 `route`，规则是让请求路径前缀为 `/` 的请求全部转发到这个 `service`。

``` bash
# 添加 service
$ curl -XPOST -H 'Content-Type: application/json' \
     -d '{"name":"example.service","url":"http://httpbin.org"}' \
     http://localhost:8001/services/

# 在 service 上添加 route
$ curl -XPOST -H 'Content-Type: application/json' \
     -d '{"paths":["/"],"strip_path":false}' \
     http://localhost:8001/services/example.service/routes
```

验证接口转发情况：

``` bash
$ curl -i http://localhost:8000/status/200
HTTP/1.1 200 OK
...
Via: kong/1.5.1
```

现在在 `example.service` 这个 `service` 开启 `demo` 插件：

``` bash
# 在 service 上开启 demo 插件
$ curl -XPOST --data "name=demo" \
       http://localhost:8001/services/example.service/plugins
```

验证插件执行情况：

``` bash
$ curl -i -H 'X-Request-Base64: aGVsbG8ga29uZw==' http://localhost:8000/status/200
HTTP/1.1 200 OK
...
X-Response-Decoded: hello kong
...
Via: kong/1.5.1
```

可以看到，请求头的 `aGVsbG8ga29uZw==`，被成功解码为 `hello kong`。

**参考：**

- 2020-05 微服务 API 网关 Kong 实践 <https://nullwy.me/2020/05/kong-gateway/>
- Plugin Development <https://docs.konghq.com/1.5.x/plugin-development/>
- Kong plugin template <https://github.com/Kong/kong-plugin>
