# 一 问题

App 安装后，第一次启动时需要登录（在某些页面提示需要登录或者直接启动在登录界面）。而只要登录成功后，就不需要每次启动时再次登录。不过，当你的 App 长期未启动，再次启动时，就需要登录。这个是怎么实现的？App 又是怎么保持登录状态的？

## 二、实现思路

客户端在登录的时候带上设备的设备号、appId，并将其作为参数传递到服务端。服务端收到参数后，使用特定的算法（例如：AES加密算法）生成 Token 和 refreshToken，返回给客户端，并且保存到 redis 缓存数据库中。服务端 redis 缓存数据库中的 Token 和 refreshToken 设置超时时长，refreshToken 超时时长大于 Token 超时时长，超时自动清除。客户端后面每次发起网络请求时服务端都要统一拦截，先取出 redis 缓存数据库中该用户对应的Token，和客户端传递的Token进行比较，如果有而且匹配则放行，如果超时没有了，客户端则带上refreshToken 调用Token刷新接口重新刷新 Token 和 refreshToken，刷新失败则进行登录认证过程。

1、客户端使用用户名跟密码请求登录，并带上设备序列号、appId；

2、服务端收到请求，去验证用户名与密码；

3、验证成功后，服务端会签发一个Token和一个refreshToken；

4、客户端收到Token和refreshToken以后必须把它们存储起来，比如放在Cookie里或者Local Storage里；

5、客户端每次向服务端请求数据的时候需要带着服务端签发的Token；

6、服务端收到请求，就去验证请求里面带着的Token，如果验证成功，就向客户端返回请求的数据；

7、客户端向服务端请求数据的时候如果Token超时，就带上 refreshToken，重新调用Token刷新接口，获取新的Token 和 refreshToken；

8、服务端收到刷新Token请求，就去验证请求里面带着的 refreshToken，如果验证成功，就向客户端返回新的Token 和 refreshToken，客户端用新的Token请求数据；

9、如果刷新 Token 失败，则重新发起登录流程。

![](D:\dev\study\minggaowang.github.io\img\login-1.jpg)

# 三 遇到的坑

## 1 同一用户并发刷新Token

由于客户端一般都是多个接口并发请求数据，当Token超时后，服务端要求客户端刷新Token，这时候客户端会多个请求并发刷新Token。当第一个接口刷新Token成功后，服务端的Token和refreshToken都发生了改变，导致后面的刷新请求验证refreshToken不通过，从而刷新失败。

解决方法：

1.客户端控制同时只有一个接口刷新Token；

2.服务端在刷新Token的时候，只产生新的Token，refreshToken值不变，只做延时；

3.服务端产生新的Token和refreshToken，但同时保存上一个老的refreshToken，新的refreshToken和老的refreshToken在设定时间内都生效。

### 2  服务端已经产生了新的Token后，但并发过来的请求还在用老的Token请求数据

解决方法：

1.服务端一律返回需要刷新，让客户端调用刷新接口。