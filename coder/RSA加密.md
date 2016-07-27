从RSA加密到HTTPS
---  
---  
今天在项目里调登录注册接口的时候，发现一个问题，由这个问题引出了下面这篇blog。  
背景是这样的：手机号和密码做为接口请求的请求参数时，密码需要进行**RSA加密**，而我在charles抓包时发现，即使是**同一个明文密码**，经过加密后得到的字串（密文）是完全不一样的。一开始我还以为是请求接口的地方出了问题，后来经过验证确实正常情况就是**每次加密的密文都不一样**，而传到服务器那边校验也能正常工作。  
出于对于这样一个校验流程的好奇，我查找了一些资料，了解了下RSA加密的过程。  
  
###非对称加密之RSA  
###RSA加密中的padding  
###加密和加签  
###RSA的应用场景:HTTPS  
###HTTP 2.0  
  
[一篇搞定RSA加密与SHA签名](http://www.kgc.cn/bbs/post/29886.shtml)  
[App安全之网络传输安全](http://mrpeak.cn/blog/encrypt/)  
[what-is-the-difference-between-the-different-padding-types-on-ios](http://stackoverflow.com/questions/5054036/what-is-the-difference-between-the-different-padding-types-on-ios)  
[iOS安全系列之一：HTTPS](http://www.oncenote.com/2014/10/21/Security-1-HTTPS/)  
[iOS安全系列之二：HTTPS进阶](http://www.oncenote.com/2015/09/16/Security-2-HTTPS2/)  
[HTTPS到底是个啥玩意儿？](https://mp.weixin.qq.com/s?__biz=MzA3MDExNzcyNA==&mid=402053009&idx=1&sn=ea531fc21a07d33f8a0408e5206c60f3)  
[HTTPS科普扫盲帖](https://segmentfault.com/a/1190000004523659?f=tt&hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)  
