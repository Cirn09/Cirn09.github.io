---
title: 🔧 从《双星物语 2》雪山入口解谜到 Plaid CSS Writeup
date: 2023-04-21 20:38:40
tags:
	- Write Up
    - 逆向
    - WIP
---

本质上是同一类问题
参考：

https://www.bilibili.com/video/BV1mx411v7P4?t=423.7&p=20
https://zhuanlan.zhihu.com/p/402823116
https://oi-wiki.org/math/number-theory/crt/



令人大开眼界的爆破解法：
https://gist.github.com/SakiiR/700d2c5ca2eddb28123493498c584189



咕，有空再写



```
4264
1: 123
2: 2
3: 23
4: 234

uid_pid.format
```

以双星物语雪山脚谜题为例 https://www.bilibili.com/video/BV1mx411v7P4?t=423.7&p=20 （因为我不玩穹，而且用这个写可以直接用到博客里）

可以看到有四个地藏：初始方向分别是`左前右左`，我们称为 ABCD 吧。

操作 A 会使 ABC 逆时针转动 90°；#$x_1$

操作 B 会使 B 逆时针转动 90°；#$x_2$

操作 C 会使 BC 逆时针转动 90°；#$x_3$

操作 D 会使 BCD 逆时针转动 90°。#$x_4$



我们不妨认为地藏朝前是 0、右是 1、后是 2、左是 3，那么初始状态就对应 `(3, 0, 1, 3)`

那么我们每次调查某个地藏都会使特定地藏的状态在模 4 下 +1，也就是说 $X^{'}=(X+1)\mod{4}$

现在是在地藏变化的视角看问题，下面把视角切换到操作的视角：操作 A 的次数记为 $x_1$，依次 BCD 分别为 $x_2, x_3, x_4$，地藏们的状态分别记为 $y_1, y_2, y_3, y_4$。

那么地藏 C 的状态就可以用这样的方程表示：$y_3=(x_1+x_3+x_4+1)\mod{4}$，其中 +1 常数项表示 C 的初始状态。

我们不妨将所有的方程都列出来，同时把系数为 0 的项也列出来（方便后续列成矩阵）

$$
\left\{
\begin{array}{**lr**}
    y_1 = x_1 + 0x_2 + 0x_3 + 0x_4 + 3 \\
    y_2 = x_1 + x_2 + x_3 + x_4 + 0 \\
    y_3 = x_1 + 0x_2 + x_3 + x_4 + 1 \\
    y_4 = 0x_1 + 0x_2 +0x_3 + x_4 + 3 \\
\end{array}
\right.
$$

解谜的目标是最终四个地藏都朝向前，整理成数学语言就是解线性同余方程组：

$$
\left\{
\begin{array}{**lr**}
    x_1 + 0x_2 + 0x_3 + 0x_4 + 3 \equiv 0 \mod(4) \\
    x_1 + x_2 + x_3 + x_4 + 0 \equiv 0 \mod(4) \\
    x_1 + 0x_2 + x_3 + x_4 + 1 \equiv 0 \mod(4) \\
    0x_1 + 0x_2 +0x_3 + x_4 + 3 \equiv 0 \mod(4)
\end{array}
\right.
$$
下面先摸了
