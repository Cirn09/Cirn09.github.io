---
title: 3步解决 ByteCTF2021 GO+Lua 逆向 - Go调试之坑与Pintool3的使用
date: 2021-10-30 19:05:46
tags:
	- Write Up
	- ByteCTF2021
typora-root-url: ..
---

如何用10分钟3步解决 ByteCTF2021 的 languages binding ？

0. 准备环境（下载、编译、开终端etc）

1. 执行下列命令，知flag长度为29（输出有删减）

   ```
   ❯ $env:GODEBUG="asyncpreemptoff=1"; py -3 .\pintool3.py -a64 -s 0xcc240 -e 0x103dab -d .\byte2021q_languages_binding\new_lang_original.exe .\byte2021q_languages_binding\new_lang_script.out
   ==========================================
   ...
   * _____________________________            | 21752
   ...
     _______________________________________  | 19150
     ________________________________________ | 19156
   > _____________________________            | 21752
   The expected input length may be 29
   ```

2. 执行下列命令，知flag为`ByteCTF{1golcwm6q_ymz7fm0df??`，猜测最后一位是`}`

   ```
   ❯ $env:GODEBUG="asyncpreemptoff=1"; py -3 .\pintool3.py -a64 -s 0xcc240 -e 0x103dab -l29 -b2 -t unique .\byte2021q_languages_binding\new_lang_original.exe .\byte2021q_languages_binding\new_lang_script.out
   ==========================================
   ...
     A____________________________ | 11112
   * B____________________________ | 11338
     C____________________________ | 11112
   ...
     }____________________________ | 11112
     ~____________________________ | 11112
   > B____________________________ | 11338
   ==========================================
   ...
     Bx___________________________ | 11338
   * By___________________________ | 11564
   ...
     B~___________________________ | 11338
   > By___________________________ | 11564
   ==========================================
   ...
     ByteCTF{1golcwm6q_ymz7fm0df}_ | 15417
     ByteCTF{1golcwm6q_ymz7fm0df~_ | 15417
   pin failed
   ```

3. 爆破倒数第2位，知flag为`ByteCTF{1golcwm6q_ymz7fm0dfx}`

总用时08:42 (CPU: AMD R7 5800H)

好，本博客结束，下面可以摸了（

<!--more-->

当了次标题党，使用`Pintool3`解这题不止3步，但10分钟内解决应该是能做到的。

## 正经解法

正经解法就是老老实实逆向、抄写、写逆，这部分不是本篇博客的重点，细节就不多讲，可以去看其他大佬的wp ，比如 v0id 大佬的 [Write up](https://bbs.pediy.com/thread-269846.htm)



首先看题目的字符串和其他特征值，可以初步判断是Go+Lua逆向。

`new_lang_script.out`大概是Lua字节码文件，肉眼猜测是xor了0x55。解密后发现文件头不太对，对着正确的文件修了下文件头、扔进官方lua解释器里还是报错，也不能正常解码字节码。猜测是换了字节码，进一步的测试和逆向也印证了我的猜想。

那么下一步就是还原字节码。v0id 还原字节码的方法比我高明多，我是bindiff+人肉bindiff找到所有指令的handler，然后根据结构体得知字节码的映射关系。

然后就可以逆出Lua部分的校验逻辑，顺着`call`指令的handler找到Go的校验代码，这道题就可以结束了。

好，本博客结束，下面又可以摸了（

---



本文尝试解决两个问题。

一是动态调试时，进程的ip经常莫名飞到一个神秘的上下文切换函数（图1），给动态调试带来了一定的困扰。

第二，观察该题的校验逻辑，Lua部分是 `return transform(flag[0])=='B' && flag[1]=='y' &&...` 模式的逐位校验，某位校验错误就会逻辑短路，直接 `return false`；Go部分也是逐位校验，但某位错误时不会break：

```python
for i in range(?):
    if transform(flag[i]) != target[i]:
        right = 0
```

唉，你看这是不是可以用pin跑。欣喜若狂的你打开了[Pintool](https://github.com/wagiro/pintool)，然后发现这个项目是py2写的，调用pin的subprocess部分对Windows Powershell兼容也不太好的样子，而且代码也不太好复用的样子。失望的你决定先手动用pin跑一下试试，试了几次的你更失望了：即使是相同的输入，两次运行icount(instruction count)也是相去甚远。多跑几次还能碰上报错。

## 解决Go动态调试的坑

首先解决第一个问题。

Go从某个版本开始加入了signal-based  asynchronous goroutine preemption（基于信号的抢占异步Goroutine），对逆向手的直观体验就是线程单步走着走着就莫名跳到了一个保存上下文的函数`asyncPreempt`。这个神秘函数的源码可以在[这里](https://github.com/golang/go/blob/fb42fb705d/src/runtime/preempt_amd64.s#L6)找到。

这个新feature也是icount不稳定的来源之一。另外 intel pin 插桩是基于 JIT 的，这个feature很可能也是pin报错的诱因（关掉之后就没再报错）。

![](/images/pintool3/go_asyncPreempt.png)

<center><div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">图1 - 神秘的asyncPreempt()</div></center>

这个feature关闭不难，源码[extern.go](https://github.com/golang/go/blob/master/src/runtime/extern.go)中可以找到相对应的介绍（以及其他调试选项的介绍）。只需要设置环境变量 `GODEBUG="asyncpreemptoff=1"`，再次调试就不会有IP乱飞的情况了。不过不建议将其设为全局环境变量，否则可能会影响同台机器上其他Go语言进程的运行效率。

## 如何Pin一个Go语言程序

很遗憾，即使关掉了signal-based  asynchronous goroutine preemption，Go依然是多线程，icount依然不稳定。

### count_range

思考一下，用户代码执行的次数应该是稳定的，带来不稳定的应该是Go的调度、信号处理之类的代码。我们可不可以只pin用户代码呢？

答案是大致可以。编译器针对代码的局部性进行了优化，使得链接的结果——函数的高低排布有一定规律可循。具体到Go语言上，`main`函数一般都在`text`段最后，向前是用户的其他函数，中间是第三方库函数，最前面是Go的核心代码。

所以我们只pin后半段的代码，应该就可以屏蔽掉Go代码带来的不稳定。intel pin可以轻松完成这个需求，加载时可以通过`IMG_IsMainExecutable`判断镜像是非为主模块，然后调用`IMG_LowAddress`和`IMG_HighAddress`可以获取主模块的内存范围。

```
pin进程空间内所有指令             <-- icount(inscount0)在这
pin主模块指令                    <-- pintools_ctf 大致在这
                                  (https://github.com/NoOne-hub/pintools_ctf)
pin主模块一段范围（用户代码）的指令  <-- 我们在这
```

好，试一下

&#123;&#123;range_count的结果&#125;&#125; (只有自带模板引擎的人才能看到图/代码)

可以看到虽然count依然不稳定，但正确输入时的count已经明显比错误输入高了。

### bcount_range

继续思考，pin的范围是不是可以继续缩减。范围越窄，受到无关代码的影响越小，Pin的效率也会提高。我们pin icount，是因为icount从一个侧面反映了程序在一个特定条件下的运行过程。从这个角度看，我们可以用基本块count代码icount，而决定基本块是否执行、执行几次是由branch指令决定的。好，我们可以用bcount(branch instruction count)代替icount。

```
pin进程空间内所有指令             <-- icount(inscount0)在这
pin主模块指令                    <-- pintools_ctf 大致在这
                                  (https://github.com/NoOne-hub/pintools_ctf)
pin主模块一段范围（用户代码）的指令  <-- 我们刚才在这
pin一段范围的branch指令           <-- 我们在这
```

好，按照这个思路写一个bcount_range试试，range就选定为`text`段靠中间随便一点到`text`段结尾吧

```
==========================================
...
  y//////////////////////////// | 21261
  z//////////////////////////// | 21261
  A//////////////////////////// | 21261
* B//////////////////////////// | 21611
  C//////////////////////////// | 21267
  D//////////////////////////// | 21259
...
==========================================
...
  Bw/////////////////////////// | 21605
  Bx/////////////////////////// | 21605
* By/////////////////////////// | 21951
  Bz/////////////////////////// | 21605
  BA/////////////////////////// | 21609
...
```

可以看到虽然count依然不稳定，但正确输入时的count已经明显比错误输入高了。

看着flag一位位被跑出来，你感到非常愉悦。但是这一切到了Go校验的部分又乱了套

```
  ByteCTF{0//////////////////// | 24031
  ByteCTF{1//////////////////// | 24030
  ByteCTF{2//////////////////// | 24031
  ByteCTF{3//////////////////// | 24031
  ByteCTF{4//////////////////// | 24031
  ByteCTF{5//////////////////// | 24031
  ByteCTF{6//////////////////// | 24033
  ByteCTF{7//////////////////// | 24027
  ByteCTF{8//////////////////// | 24031
  ByteCTF{9//////////////////// | 24033
  ByteCTF{{//////////////////// | 24029
  ByteCTF{_//////////////////// | 24029
  ByteCTF{}//////////////////// | 24035
  ByteCTF{a//////////////////// | 24031
  ByteCTF{b//////////////////// | 24031
  ByteCTF{c//////////////////// | 24033
  ByteCTF{d//////////////////// | 24035
  ByteCTF{e//////////////////// | 24035
  ByteCTF{f//////////////////// | 24031
  ByteCTF{g//////////////////// | 24035
  ByteCTF{h//////////////////// | 24031
  ByteCTF{i//////////////////// | 24029
  ByteCTF{j//////////////////// | 24027
  ByteCTF{k//////////////////// | 24031
  ByteCTF{l//////////////////// | 24031
  ByteCTF{m//////////////////// | 24037
  ByteCTF{n//////////////////// | 24029
  ByteCTF{o//////////////////// | 24031
  ByteCTF{p//////////////////// | 24035
  ByteCTF{q//////////////////// | 24033
  ByteCTF{r//////////////////// | 24033
  ByteCTF{s//////////////////// | 24031
  ByteCTF{t//////////////////// | 24031
  ByteCTF{u//////////////////// | 24037
  ByteCTF{v//////////////////// | 24029
  ByteCTF{w//////////////////// | 24029
  ByteCTF{x//////////////////// | 24031
  ByteCTF{y//////////////////// | 24031
  ByteCTF{z//////////////////// | 24029
  ByteCTF{A//////////////////// | 24031
  ByteCTF{B//////////////////// | 24029
  ByteCTF{C//////////////////// | 24029
  ByteCTF{D//////////////////// | 24029
  ByteCTF{E//////////////////// | 24033
  ByteCTF{F//////////////////// | 24031
  ByteCTF{G//////////////////// | 24033
  ByteCTF{H//////////////////// | 24033
  ByteCTF{I//////////////////// | 24031
  ByteCTF{J//////////////////// | 24027
  ByteCTF{K//////////////////// | 24033
  ByteCTF{L//////////////////// | 24031
  ByteCTF{M//////////////////// | 24031
  ByteCTF{N//////////////////// | 24035
  ByteCTF{O//////////////////// | 24043
  ByteCTF{P//////////////////// | 24043
* ByteCTF{Q//////////////////// | 24045
  ByteCTF{R//////////////////// | 24039
  ByteCTF{S//////////////////// | 24039
  ByteCTF{T//////////////////// | 24037
  ByteCTF{U//////////////////// | 24035
  ByteCTF{V//////////////////// | 24039
  ByteCTF{W//////////////////// | 24037
  ByteCTF{X//////////////////// | 24035
  ByteCTF{Y//////////////////// | 24033
  ByteCTF{Z//////////////////// | 24031
```

上边是一次运行的结果，在运行一次会得到不同的结果。可以看到，最大值也不过比次高大2而已。

这是什么回事？我们再细看一下Go校验部分的汇编

![gocheck](/images/pintool3/gocheck.png)

<center><div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">图2 - Go校验函数片段</div></center>

哦，无论输入对错，branch指令数量都是固定的，唯一的区别只有`jz`跳转与否。我们可以退回到icount，正确与否有一个`xor`指令的差别。但一条指令的差异太小，会被Go自己代码的影响覆盖。

### nbcount_ragne

那我们继续延伸思路，只要不停下脚步，道路就会不断延伸，所以，不要停下来啊……

确实有这种branch指令数量固定的情况，只有跳转与否的区别。刚好pin也提供了跳转时插桩的选项`IPOINT_TAKEN_BRANCH`，但似乎没给不跳转时插桩。没关系，跳不跳都count++，然后加一个跳转时count\-\-，这样结果就是未跳转的branch数了。

```
pin进程空间内所有指令                <-- icount(inscount0)在这
pin主模块指令                       <-- pintools_ctf 大致在这
                                     (https://github.com/NoOne-hub/pintools_ctf)
pin主模块一段范围（用户代码）的指令     <-- 我们之前在这
pin一段范围的branch指令              <-- 我们刚才在这
pin一段范围的(not) taken branch指令  <-- 我们在这
```

再来实验一下。taken branch count和bcount没太大区别，就不分析了。这儿只贴not taken branch count

```
==========================================
...
  x//////////////////////////// | 10783
  y//////////////////////////// | 10783
  z//////////////////////////// | 10783
  A//////////////////////////// | 10783
* B//////////////////////////// | 11009
  C//////////////////////////// | 10783
  D//////////////////////////// | 10783
...
  Bw/////////////////////////// | 11009
  Bx/////////////////////////// | 11009
* By/////////////////////////// | 11235
  Bz/////////////////////////// | 11009
  BA/////////////////////////// | 11009
......
  ByteCTF{0//////////////////// | 12591
* ByteCTF{1//////////////////// | 12590
  ByteCTF{2//////////////////// | 12591
  ByteCTF{3//////////////////// | 12591
...
  ByteCTF{1e/////////////////// | 12590
  ByteCTF{1f/////////////////// | 12590
* ByteCTF{1g/////////////////// | 12589
  ByteCTF{1h/////////////////// | 12590
...
```

不得了，count稳定了，而且Go校验的部分可以跑了。

看着flag一位位被跑出来，你感到非常愉悦。但是这一切到了倒数第二位又乱了套，不过这次问题不大，爆破1位输入具体原因就不分析了，有兴趣的可以自己看Lua字节码。



## 如何13步解决 ByteCTF2021 GO+Lua 逆向

1. 下载题目
2. 使用ida打开题目
3. 将Image base调整为0 (Alt+esr, 0, enter)
4. 程序`text`中间随便选个地址，复制下来
5. `text`尾地址复制下来
6. 打开Terminal
7. `git clone https://github.com/Cirn09/pintool3`
8. 编译`bcount_range.cpp`
9. 编辑`pintool3.conf`
10. `$env:GODEBUG="asyncpreemptoff=1"; py -3 .\pintool3.py -a64 -s 0xcc240 -e 0x103dab -d new_lang.exe new_lang_script.out` 得到flag长度为29
11. `$env:GODEBUG="asyncpreemptoff=1"; py -3 .\pintool3.py -a64 -s 0xcc240 -e 0x103dab -l29 -b2 -t unique new_lang.exe new_lang_script.out` 得到flag片段
12. 爆破倒数第二位，得到完整flag
13. 提交flag

## Pintool3

以上我们探索的结果都在[Pintool3](https://github.com/Cirn09/pintool3)项目里，欢迎star、fork。

### 命令行接口

使用逻辑大致和[Pintool](https://github.com/wagiro/pintool)相似：

`-a`指定架构，只支持32和64；

`-b`指定pin的类型：`0`是所有branch指令，`1`是taken branch，`2`是not taken branch；

`-s`、`-e`分别指定范围的起始和结束；

`-m`可以指定要pin的模块名字，考虑到可能要pin dll；

`-c`指定爆破使用的字符集，默认是`string.printable - string.whitespace`；

`-p`指定未知部分的填充字符，默认是`_`；

`-k`告诉`Pintool3`已知的flag片段，比如`ByteCTF{/////////_ymz7fm0dfx}`，不知道的部分用`-p`指定的padding填充；

`-d` `Pintool3`会尝试探测flag长度；

`-l`是flag长度，当`-d`开启时是尝试探测的最大长度；

`-t`是如何在count中选定正确输入，可以max/min，也可以unique选定唯一值，这个选项没有`Pintool`丰富，因为`Pintool3`默认多线程，不会中途选定停止；

`-o`是爆破的顺序，除了`normal`和`reverse`之外，还有一个用于乱序的`detect` ；

`-r`指定重试次数，有的程序在pin时就是有概率会报错；

`--disable-multiprocess`可以关掉多进程。

### 编程接口

`-o`、`-t`肯定没有覆盖所有的情形、所有的需求，额外的需求可以通过编程拓展，把`Pintool3`当作包导入。

`Pintool3`最主要的两个函数是`pin`和`multipin`，这两个函数的不同只在参数和返回值。

- `pin`需要一个输入，执行一次`bcount_range`，返回一个`Pinfo`对象，包含`bcount_range`的`stdout`、`stderr`和`count`信息。

- `multipin`接受一个输入数组，返回一个`Pinfo`数组。

## PS

`bcount_range`大概依赖于`range`的选定，not taken branch count突然稳定可能是运气好。更科学的方法是以相同条件多跑几次程序进行统计，然后屏蔽掉执行不稳定的branch。

顺便一提，比赛时和室友Kotori一起看，然后我们一起把两个加减号搞反了，直到比赛结束才发现。。。
