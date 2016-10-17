从RSA加密说起
---  
---  
今天在项目里调登录注册接口的时候，发现一个问题，由这个问题引出了下面这篇blog。  

背景是这样的：手机号和密码做为接口的请求参数时，密码需要进行**RSA加密**，而我在charles抓包时发现，即使是**同一个明文密码**，经过加密后得到的字串（密文）每次都是完全不一样的。一开始我还以为是请求接口的地方出了问题，后来经过验证确实正常情况就是**每次加密的密文都不一样**，而传到服务器那边校验也能正常工作。  

出于对于这样一个校验流程的好奇，我查找了一些资料，了解了下RSA的相关内容。  

### 非对称加密之RSA  
为保障传输数据的安全，需要使用加密算法。加密算法一般分为两类：“对称”加密算法和“非对称”加密算法，常用的分别为**AES**和**RSA**。“对称”加密算法的安全性完全取决于对加密密钥的管理，不在本文的关注范围内。

AES加密算法的原理可参考[App安全之网络传输安全](http://mrpeak.cn/blog/encrypt/)  

不同于“对称”加密算法，“非对称”加密算法实现了在不直接传递密钥的情况下完成解密。具体流程如下：

>1. 乙方生成两把密钥（公钥和私钥）。公钥是公开的，任何人都可以获得，私钥则是保密的。
>2. 甲方获取乙方的公钥，然后用它对信息加密。
>3. 乙方得到加密后的信息，用私钥解密。

如果公钥加密的信息只有私钥解得开，那么只要私钥不泄漏，通信就是安全的。这种算法的破解难度与密钥的长度正相关。一般来说，1024位的RSA密钥基本安全，2048位的密钥极其安全。

RSA算法的原理可参考[RSA算法原理](http://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)

关于RSA这种非对称加密算法，在App的使用当中，需要明白其主要作用有2个：

- 信息加密：通信双方可以在公开的网络环境下，“安全”的商量对称加密算法所使用的密钥。
- 电子签名：为了防止中间人攻击，通信双方在商量密钥之前可以通过签名算法确认对方的身份。

非对称加密算法本身是一种加密算法，但由于RSA本身加解密的性能在现在的计算机硬件条件下存在一定瓶颈，同时对加密数据的“安全长度”也有限制，**被加密数据的长度一般要求不超过公钥的长度**。所以RSA更多的是被用来商量一个密钥，如果密钥是安全的，那么后续的通信都可以使用上面提到的AES来完成，AES在性能上不存在瓶颈。

### RSA加密中的padding
padding即填充方式，由于RSA加密算法中要加密的明文是要比模数小的，padding就是通过一些填充方式来限制明文的长度。

- RSA_PKCS1_PADDING 填充模式，最常用的模式  
输入：必须 比 RSA 钥模长(modulus) 短至少11个字节, 也就是　RSA_size(rsa) – 11 如果输入的明文过长，必须切割，然后填充。  
输出：和modulus一样长
根据这个要求，对于1024bit的密钥，block length = 1024/8 – 11 = 117 字节

- RSA_PKCS1_OAEP_PADDING  
输入：RSA_size(rsa) – 41  
输出：和modulus一样长

- RSA_NO_PADDING　　不填充  
输入：可以和RSA钥模长一样长，如果输入的明文过长，必须切割，　然后填充  
输出：和modulus一样长  

研究到这里，再一看项目里加密的代码，终于明白了，我们用了**RSA_PKCS1_PADDING**模式！其中切割出来的11字节用随机数填充，从而每次加密后的密文都完全不一样！  

加密算法如下：  

    - (NSData *)encrypt:(NSString *)plainText usingKey:(SecKeyRef)key error:(NSError **)err
    {

        size_t cipherBufferSize = SecKeyGetBlockSize(key);

        uint8_t *cipherBuffer = NULL;

        cipherBuffer = malloc(cipherBufferSize * sizeof(uint8_t));

        memset((void *)cipherBuffer, 0*0, cipherBufferSize);

        NSData *plainTextBytes = [plainText dataUsingEncoding:NSUTF8StringEncoding];

        unsigned long blockSize = cipherBufferSize - 12;

        int numBlock = (int)ceil([plainTextBytes length] / (double)blockSize);

        NSMutableData *encryptedData = [[NSMutableData alloc] init];

        for (int i=0; i<numBlock; i++) {

            unsigned long bufferSize = MIN(blockSize,[plainTextBytes length] - i * blockSize);

            NSData *buffer = [plainTextBytes subdataWithRange:NSMakeRange(i * blockSize, bufferSize)];

            OSStatus status = SecKeyEncrypt(key, kSecPaddingPKCS1,
                                        (const uint8_t *)[buffer bytes],
                                        [buffer length], cipherBuffer,
                                        &cipherBufferSize);

            if (status == noErr)
            {
                NSData *encryptedBytes = [[NSData alloc]
                                       initWithBytes:(const void *)cipherBuffer
                                       length:cipherBufferSize];
                [encryptedData appendData:encryptedBytes];
            }
            else
            {
                if (err)
                {
                    *err = [NSError errorWithDomain:@"errorDomain" code:status userInfo:nil];
                }
                free(cipherBuffer);
                return nil;
            }
        }

        if (cipherBuffer)
        {
            free(cipherBuffer);
        }

        return encryptedData;
    }


### RSA签名验证  
在网络中传输数据时，为了防止中间人篡改信息，通讯双方可以通过非对称签名算法确认对方的身份，这也即RSA签名验证。  

JSPatch在传js脚本时就用了这个方法保证传输的安全。校验过程如下:  
![  ](https://github.com/simonleo/devlog/blob/master/sources/JSPatchSecurity.png?raw=true)  

第一步在服务端计算脚本文件的MD值，做为这个文件的数字签名，用存在服务端的私钥对这个MD5值进行加密。然后把这个MD5值和脚本一起打包下发给客户端。客户端拿到脚本和加密后的MD5值，用存在客户端的公钥进行解密，拿到服务端计算出来的MD5值，本地再对脚本文件计算一遍MD5值，对比这两个值是否一致，若一致，则说明传输过程中数据没有被篡改。  

需要注意的是，这个校验的目的，是防止数据传输时被篡改，对于数据内容泄露不是太在意。第三方截获请求想要篡改下发恶意脚本时，需要用私钥加密这个脚本的MD5值一起下发，才能最终在客户端通过验证，只要第三方没有私钥，就不能达到目的。

校验算法如下：  

    -(BOOL)verifyTheDataSHA1WithRSA:(NSData *)signature andplainText:(NSString*)plainText
    {
        NSData *sigdata = signature;
        SecKeyRef publicKeyRef= [self getPublicKey];
        size_t signedBytesSize = SecKeyGetBlockSize(publicKeyRef);

        OSStatus status = SecKeyRawVerify(publicKeyRef, kSecPaddingPKCS1SHA1,
                                          (const uint8_t *)[[self getHashBytes:[plainText dataUsingEncoding:NSUTF8StringEncoding]] bytes],
                                          kChosenDigestLength,
                                          (const unsigned char *)[sigdata bytes],
                                          signedBytesSize);
        return status == noErr;
    }  

>注：openssl 中  
>公钥加密 = 加密  
>私钥解密 = 解密  
>私钥加密 = 签名  
>公钥解密 = 验证  

### 关于HTTPS  
HTTPS应该是RSA应用的最重要的场景，里面同时涉及了“加密”和“签名”的内容。原本以为在这篇里一并把https也总结一下，看着看着发现https涉及的东西实在是太多了，想要顺便提一下实在是不自量力，还是以后专门学习一下再写一篇吧～  

HTTPS相关的文章先贴一下，后面再看：  
[iOS安全系列之一：HTTPS](http://www.oncenote.com/2014/10/21/Security-1-HTTPS/)  
[iOS安全系列之二：HTTPS进阶](http://www.oncenote.com/2015/09/16/Security-2-HTTPS2/)  
[HTTPS到底是个啥玩意儿？](https://mp.weixin.qq.com/s?__biz=MzA3MDExNzcyNA==&mid=402053009&idx=1&sn=ea531fc21a07d33f8a0408e5206c60f3)  
[HTTPS科普扫盲帖](https://segmentfault.com/a/1190000004523659?f=tt&hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)

本文学习参考了以下文章：  
[App安全之网络传输安全](http://mrpeak.cn/blog/encrypt/)  
[RSA算法原理](http://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)  
[一篇搞定RSA加密与SHA签名](http://www.kgc.cn/bbs/post/29886.shtml)  
[what-is-the-difference-between-the-different-padding-types-on-ios](http://stackoverflow.com/questions/5054036/what-is-the-difference-between-the-different-padding-types-on-ios)  
[JSPatch 部署安全策略](http://blog.cnbang.net/tech/2879/)
