# Request Header Too Large

## 问题分析

在 CSMS 系统需求联调阶段，发现许多页面请求返回大量 400 错误。

![Img](./FILES/Request%20Headers%20Too%20Large.md/img-20241113112746.png)

起初，以为是服务部署造成的问题，但后端团队确认服务部署正常，且其他用户访问无类似问题。经过日志分析，发现 400 错误的原因是请求头过大。

![Img](./FILES/Request%20Headers%20Too%20Large.md/img-20241113142621.jpg)

通过对比开发环境和测试环境，我注意到开发环境中 Authorization 请求头显著偏大。

![Img](./FILES/Request%20Headers%20Too%20Large.md/img-20241113142828.jpg)

![Img](./FILES/Request%20Headers%20Too%20Large.md/img-20241113142833.jpg)

进一步分析发现，开发环境的登录账户配置了十多个角色，导致生成的 token 体积过大。

此外，请求中携带的 Cookie 也很大，包含神策的两个下发 token 以及一个未知来源的 token。

![Img](./FILES/Request%20Headers%20Too%20Large.md/img-20241113143041.png)

经过解析，后端团队确认该未知 token 并非由 CSMS 系统下发，它包含 user-info 和 user-info-base64 等用户信息。这些累积导致请求头超出 Tomcat 默认允许的最大大小 8KB（8192 字节），从而触发 400 错误。

![Img](./FILES/Request%20Headers%20Too%20Large.md/img-20241113143304.png)

## 解决方案

1. 修改网关设置，移除 user-info 请求头的插入，仅保留 user-info-base64 请求头。user-info 信息在系统升级后不可用，参考：https://confluence.autel.com/pages/viewpage.action?pageId=268009524
2. 在 Nacos 配置的 BASE_GROUP 下，更新 request-setting.yml 文件，将请求头大小限制调整为 32KB。此配置更新将适用于所有引用该配置的后台微服务。

```yaml
- data-id: request-setting.yml
  group: BASE_GROUP
  refresh: true
```

或者在自己的服务配置文件中增加如下配置：

```yaml
server:
  max-http-header-size: 32768
```

## 参考

配置 max-http-header-size 需要根据你的具体应用需求和部署环境进行权衡。设置合适的值可以在提升用户体验和应用性能的同时减少安全风险。下面是一些考虑因素和推荐的设置范围，可以帮助你决定合适的 max-http-header-size 值。

### 考虑因素

1. 应用类型：

- 如果你的应用需要处理较大的 HTTP 头信息（例如，包含大量自定义头部、复杂的 Cookie、JWT Token 等），可能需要更大的 max-http-header-size。
- 如果你的应用头信息较为简单，可以保持较小的值，这样可以增强安全性和性能。

2. 安全性：

- 较大的值可能会增加拒绝服务（DoS）攻击的风险，因为恶意请求可以发送庞大的头信息消耗服务器资源。
- 确保你在调整 max-http-header-size 的同时，使用其他安全措施（如请求速率限制、IP 黑白名单等）来缓解安全风险。

3. 性能和资源限制：

- 较大的头信息会占用更多的内存和计算资源，尤其是在高并发环境下。
- 考虑服务器的内存和 CPU 配置，避免过度消耗资源。

4. 兼容性：

- 保留合理的头部大小配置，确保客户端和代理服务器能处理，不会因为头信息过大而导致请求失败。

### 推荐设置范围

在大部分应用中，以下设置范围是合理的：

- 8KB - 16KB：
  - 适合大多数常见的 Web 应用，确保能够处理标准的 HTTP 头信息。
  - 与大多数默认配置相近（如 Tomcat）。
- 32KB - 64KB：
  - 如果你的应用有具体需求，需要处理较多或较大的头信息，可以增大此值。
  - 适用于需要处理多重认证头、复杂的 Cookie 或者其他自定义头部较多的情况。
- 128KB 及以上：
  - 通常不建议设置为如此大的值，除非有非常明确的需求。
  - 这种配置可能会带来性能和安全问题，应结合其他安全措施。
  - 需要确保应用和服务器能够有效处理这么大的头信息。
