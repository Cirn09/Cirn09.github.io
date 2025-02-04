---
title: "RealWorld2021 Write Up: Personal Proxy"
date: 2021-01-11 15:48:47
tags:
    - Write Up
    - Real World
typora-root-url: ..
---

Personal Proxy一题的Padding Oracle attack解法，是非预期解法。

我觉得比官方解有趣。

<!--more-->

周末跟着队伍打了Real World 2021，太菜了只做了Personal Proxy一道题。赛后发现我的做法是非预期，虽然比官解麻烦，但是方法比官解有趣一些，所以决定来水篇博客。

# 背景知识

## socks5

socks5是一种网络传输协议，一般将其用于代理。socks5流程分四步：选择认证方式、认证、选择连接目标、隧道传输，其中第二步可选。
具体细节参考 [Wiki](https://zh.wikipedia.org/wiki/SOCKS#SOCKS5)、[RFC 翻译](https://www.quarkay.com/code/383/socks5-protocol-rfc-chinese-traslation)，攻击只用了第一步，简单看一下选择认证方式的交互流程即可。

## AES-CFB

![](/images/rwctf2021/aes-cfb.png)

CFB的特点是不会直接把明文扔进AES进行加密，而是在第一轮把IV扔进AES进行加密，得到的输出与明文异或得到密文。之后的第$n$轮会把第$n-1$轮的密文扔进AES，得到的输出与明文异或得到密文。

为了方便，我们**将第$i$轮AES的输出称为"轮密钥 ($RoundKey_i$)"**，轮密钥与明文异或得到密文。显然，轮密钥由key和上轮密文决定（第一轮除外），当key固定时，可认为轮密钥仅与上轮密文相关。一组密文对应唯一一组下一轮密钥，为了方便简称为密文-密钥对。
$$
RoundKey_{i+1} = Func(Cipher_{i}) (i>1) \\\\
	Cipher_i = Plain_i \oplus RoundKey_i
$$

# 题目信息

![](/images/rwctf2021/rwpp_info.png)

附件提供了proxy server的Dockerfile，以及一段简短的proxy client与server交互的流量包。

```dockerfile
# Dockerfile

...
RUN mkdir -p /server &&\
    cd /server &&\
    wget --no-check-certificate https://github.com/snail007/shadowtunnel/releases/download/v1.7/shadowtunnel-linux-amd64.tar.gz &&\
    tar xvf shadowtunnel-linux-amd64.tar.gz &&\
    rm shadowtunnel-linux-amd64.tar.gz
...
CMD /server/run.sh
...
```

```shell
# run.sh
#!/bin/bash

danted -D -d 2 -f /server/danted.conf

echo "shadowtunnel password: $PASSWORD"
./shadowtunnel -e -f 127.0.0.1:61080 -l :50000 -p $PASSWORD

```

从Dockerfile可以得知，server使用[shadowtunnel](https://github.com/snail007/shadowtunnel/)部署。

shadowtunnel是一个很简单的proxy server。

通过阅读源码可以知道它默认使用aes-192-cfb对流量进行加密，对上传和下载的加密过程各自独立，但密钥和iv都是相同的。

 结合cfb的流程可知，**上传和下载第一轮轮密钥是相同的**。



学长看了流量，觉得从长度上看很像socks5。我也看了看，觉得确实如此。

![](/images/rwctf2021/rwpp_netcap.png)

# 解题步骤

目前并没有针对AES的有效攻击，所以直接恢复key和iv是不现实的，可行的思路只有恢复流量包每一轮的密钥，或者说恢复流量包中所有的密文-密钥对。

##  猜解第一轮密钥

因为send和recv的$RoundKey_1$是相同的，而且socks5握手过程中部分字段对比较固定，所以我们可以把第一轮密钥猜解出来。

```
send_cipher: 78 05 cb a2|09 2b 82 ce eb 89 06 0a e0 6c|7e c2
send_plain:  05 02 00 01|05 01 00 01 c0 a8 1f 20 1f 40|50 4f

RoundKey1:   7d 07 cb a3 0c 2a 82 cf 2b 21 19 e5 ff 2c 2e 8d

recv_cipher: 78 07|ce a3 0c 2b 2e da 2b 23 9c f1|b7 78 7a dd
recv_plain:  05 00|05 00 00 01 ac 15 6c bf c5 d8|48 54 54 50
```

每个字段的语义可以去参考上边的[链接](#socks5)。

我们这里只需关注send和recv的第一个包，为了方便简称为s1和r1。

s1的第一个字节是固定的版本号，第二个字节声明了METHODS也就是这个包剩下的长度，剩下的部分是客户端支持的认证方式。

## 构造握手包的测试

根据我们解密出来的第一组明文，可以知道socks5 server选择了了认证方式0，也就是无需认证。那他接不接受其他认证方式呢？验证一下

```python
class Enc:
    key = list(binascii.unhexlify('7d07cba30c2a82cf2b2119e5ff2c2e8d'))
    i = 0

    def __init__(self, key2=None):
        if key2:
            self.key += list(binascii.unhexlify(key2))

    def enc(self, input):
        out = []
        for k, x in zip(self.key[self.i:], input):
            out.append(k^x)
        self.i += len(input)
        return bytes(out)

def test(e, payload):
    e.i = 0
    p = remote('13.52.88.46', 50000)
    p.send(e.enc(payload))
    return p.recv()

e = Enc()
print(test(e, b'\x05\x01\x00'))
print(test(e, b'\x05\x01\x02'))

# b'x\x07'
# b'x\xf8'
```

可以重复测试验证，socks5 server只接受认证方式0

进一步

```
print(test(e, b'\x05\x04\x01\x02\x03\x04'))
print(test(e, b'\x05\x04\x01\x02\x03\x00'))
# b'x\xf8'
# b'x\x07'
```

## 恢复第一轮密钥的方法

现在攻击思路已经很明显了，我们只需要知道最开始两字节的密钥，就可以构造握手包依字节爆破密钥：向服务器发送"enc(05 + length + 非0*(length-1)) + 测试密文"，如果服务器返回'\x78\x07'，则测试密文对应的明文就是00，对应的密钥自然也就是测试密文。

但是，因为AES-CFB的特性，第一轮密文影响第二轮密钥，为了恢复第一轮密文-第二轮密钥对，我们只能把length字段设成和流量包中相同的2，如此一来就无法施展这种oracle攻击。

所以这种思路只能恢复第一轮密钥，而第一轮密钥我们之前就已经推理出来了，这个攻击方法好像没什么卵用。。。

## 恢复任意密文-密钥对的方法

不然，我们知道只要固定了第一轮密文，第二轮密钥就是确定的。知道第二轮密钥就可以任意构造第二轮密文，进而爆破第三轮密钥，这种方法就可以恢复出所有的密文-密钥对。

### 细节和优化

恢复第三轮密钥时要注意第二轮明文不能出现0x00；

可以先将整轮16个字节，所有位置可能的密钥先筛选出来，然后每个位置逐个实验（就是每轮先把0000000、01010101、02020202...都试一遍，可以筛选出所有可能的密钥，然后逐字节测试）；



[最后的脚本](https://gist.github.com/Cirn09/82f21ffcd9bb8db4917812782e2186e1)

# 预期解

[官解链接](https://gist.github.com/zTrix/59ad1ab7a95074e66ecef8a497b25b30)

预期解要比这种oracle简单，猜解出第一轮密钥之后就可以连接proxy server，让proxy server连接自己的socks5 server，socks5 server收到的都是明文，剩下的就为所欲为了。

做题时为什么错过了简单的官解呢，因为当时只测试了连接1.1.1.1和8.8.8.8，回应都是0x06: TTL expired，所以就以为它没法主动连外网。。。。

刚好我也没有外网IP
