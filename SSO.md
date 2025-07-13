# 起因

每个服务都有自己的一套独立认证系统，当用户在应用之间进行跳转时很麻烦。

# session机制

session 是一种保存上下文信息的机制。面向用户并保存在服务端中。

每次收到客户端请求的时候，都会判断请求中的session

对每次 http 请求，都经历以下步骤处理：

- 服务端首先查找对应的 cookie 的值（sessionid）。
- 根据 sessionid，从服务器端 session 存储中获取对应 id 的 session 数据，进行返回。
- 如果找不到 sessionid，服务器端就创建 session，生成 sessionid 对应的 cookie，写入到响应头中。
- 客户端保存sessionid

session 是由服务端生成的，并且以散列表的形式保存在内存中。

集群环境下的session共享方案分<b>复制</b>和<b>集中存储</b>。

# SSO

SSO即单点登录（Single Sign On）。

## CAS方案

CAS即认证服务中心（Central Authentication Service），不是compare and set。

对于完全不同域名的系统，cookie 是无法跨域名共享的，因此 sessionId 在页面端也无法共享，因此需要实现单店登录，就需要启用一个专门用来登录的域名如（ouath.com）来提供所有系统的 sessionId。

1. 进入A服务，未登录，跳转OUATH页面登录。
2. OUATH认证登录，种cookie到OUATH域名（下次B服务登录时不需要再认证）。并在服务端存储一个ticket和sessionId的键值对。
3. 带着ticket重定向到A页面。
4. A用ticket到OUATH服务做session同步，种cookie给自己，原地重定向登录成功。

### 扩展

#### OAUTH2

OAUTH2是三方授权协议。

A、B认证系统独立，A授权给B，使用B的认证信息可以访问A权限范围内的资源。