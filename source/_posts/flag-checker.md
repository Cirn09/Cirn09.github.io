---
title: zer0pts 2022 Flag-Checker Write Up
date: 2022-03-23 17:04:02
tags:
	- Write Up
	- zer0pts
typora-root-url: ..
---

## 前言

有幸参加了zer0ptsCTF2022，有幸在一开始就抽到了本次比赛最折磨的题目，经过两天的鏖战（坐牢），终于在比赛最后4分钟拿到该题一血，成功出狱。

![pepe](/images/flag-checker.assets/pepe.jpg)

<center><div style="color:orange;
    display: inline-block;
    color: #999;
    padding: 2px;">从一位大概是越南<a style="color:" href="https://github.com/lanleft/CTF_Writeups/tree/master/8_zer0pts22_A-part-of-Flag-Checker">友人</a>那偷来的非常贴切的表情包</div></center>


## 题目结构

### 概览

本题目由3个 exe 文件组成，彼此间使用命名管道`\\.\pipe\anime`通信，用进程的父子关系模拟二叉树，以用户输入的 flag 字符串构造满二叉树，最后层序遍历二叉树验证用户输入是否正确。

3个 exe 如下：

1. `task.exe`: 所有进程的父进程，进程二叉树的父节点。负责用户交互、启动`anime.exe`和`subprocess.exe`。使用`autoit`语言编写；
2. `anime.exe`: 控制进程，负责构造进程二叉树和验证二叉树。使用`C++`编写；
3. `subprocess.exe`: 子孙进程，进程二叉树的非父节点。使用`Go`语言编写。

### `autoit`逆向

`aotuit`是个解释型语言，提供了编译为 exe 的功能。通过替换资源就可以确认，它生成的 exe 实际上就是把脚本扔进 exe 的资源里。

![image-20220322170348495](/images/flag-checker.assets/image-20220322170348495.png)

<center><div style="border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">autoit 生成的 exe 的资源</div></center>


这个语言可以用这个[工具](http://domoticx.com/autoit3-decompiler-exe2aut/)简单反编译。不过这个工具不支持 x64 的 exe ，这题提供的 exe 刚好是 x64（出题人的恶意*1）。

反编译出来的代码完美还原了符号信息，还原了出题人的恶意*2：令人误解的符号。代码里充满了像`nullptr`、`inullptr`、`$unidentified_flying_object`之类的变量名，名叫`$runtime_newproc`、实际上是`GetExitCode Process`的函数，以及`getanimelist`、`getanimedescription`、`findbestanimefrommyanimelist`这种让我真的怀疑这程序是不是真的访问了[MyAnimeList.net](https://myanimelist.net/)（我甚至真的去 MyAnimeList看了下bestanime。

### 通讯协议

为了下边解释方便，这里直接把进程间的通讯协议贴上来。

下文的整理以`anime.exe`为主视角，其中`signal0`表示`anime.exe`接收到的信息类型 0，`respond0`表示`anime.exe`对`signal0`的回应消息。各字段默认 4 字节，非 4 字节的会用`@size`标注。

```
signal0 {
    0,
    subPid, subTid,
    unuse
}
respond0 { 0x0000DEAD }

signal1 {
    1,
    char
}
respond1 { 0x0000C0DE }

signal2 {
    2,
    subPid, subTid
}
respond2 {
    0x12345678,
    isLeadNode
}

signal3 {
    3,
    id
}
respond3 {
    0x0BADFOOD, padding,
    address@8, size,
    task.pid, task.tid
}

signal4 {
    4,
    subPid, subTid,
    padding,
    address@8, size,
    data,
    ppid, ptid
}
respond4 { 0xCAFEBABE }

signal5: undefined

signal6 {
    6,
    subPid, subTid
}
respond { 6 } // 实际上是signal6的内容未作修改，原地返回

signal7 {
    7,
    pipeHandle1, pipeHandle2
}
respond7 { result }
```



### 工作流程

整体的流程可以以如下类 Python 代码概括：

```python
flag = input()
flag = flag.ljust(110, random)
check_sum = 0
for c in flag:
    check_sum += check(c)
if check_sum == 0:
    print('Well done. You got the flag!')
else:
    print('Oops. try again!')
```

#### 1. 准备

首先`task`两次弹窗，获取用户输入。这两次弹窗点“No/Cancel”的话，会弹[Youtube](https://www.youtube.com/watch?v=dQw4w9WgXcQ)，具体是啥可以自己去看看（出题人的恶意*3。

![image-20220322150953785](/images/flag-checker.assets/image-20220322150953785.png)

<center><div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">两次弹窗及相应源码</div></center>

然后`task`会将用户输入转换为字节数组，并将用户输入随机填充到长度为 110为止（所以如果运气足够好的话，你什么都不输，程序也会弹窗正确）。

启动`anime`子进程。`anime`子进程会将父进程也就是`task`的`pid`保存，用于后续读取`task`的内存。



伪代码抄写：

```python
task.input = input('Please enter the flag').ljust(110, random)
anime.ppid = task.pid
```

#### 2. 发送Flag

`signal1`: `task`将用户输入字符串的一个字节发送给`anime`，`anime`将其保存到全局变量`string input`。



伪代码抄写：

```python
idx_input = 0
anime.input += task.input[idx_input++]
```

#### 3. 查找资源地址

`signal3`: `anime`查找`task`的资源`subprocess`的地址和大小。

当然，这里的地址属于`task`的内存空间。

`task.exe`有个名为`RCData/Scr1pt`的资源(resource)，这个资源以某种数据结构保存了三个加密的`exe`。下面是dump资源并解密的脚本。

```python
from Crypto.Cipher import AES
import hashlib
import struct

def dec(data:bytes):
    s = b'abO4oHxrfR03YwaX4KuEFUoV'
    key = hashlib.sha256(s).digest()
    aes = AES.new(key[:16], AES.MODE_CBC, iv=b'\x00'*16)
    return aes.decrypt(data)


if __name__ == '__main__':
    with open('task.exe', 'rb') as f:
        data = f.read()
    magic = b'\xad\xde\xfe\xca\x0d\xf0\xad\x0b'
    start = data.index(magic)
    size = struct.unpack('Q', data[start+8:start+0x10])[0]
    s = start + 0x10
    for i in range(size):
        chunk_id = struct.unpack('Q', data[s:s+0x8])[0]
        chunk_size = struct.unpack('Q', data[s+0x8:s+0x10])[0]
        print(hex(chunk_id))
        print(hex(chunk_size))
        d = data[s+0x10:s+0x10+chunk_size]
        de = dec(d)
        with open(f'{chunk_id:08x}.exe', 'wb') as f:
            f.write(de)
        s += 0x10 + chunk_size
```

`Scr1pt`中实际保存了3个exe，用到的只有`id`为`0xfff01870`一个，我们把它称为`subproc.exe`。

那么其他两个是什么呢？一个是同样使用`autoit`编写的假flag（出题人的恶意*4：

```vbscript
Func compute()
	Local $msg = ""
	For $i = 0 To 40 Step +1
		Sleep(10000)
		$msg = $msg & Chr(getchar($i))
	Next
	Return $msg
EndFunc

Func getchar($number)
	If $number <= 9 Then
		Return 48 + $number
	Else
		Local $sum = 0
		For $i = 1 To 10 Step +1
			$sum = BitAND($sum + getchar($number - $i), 255)
		Next
		Return $sum
	EndIf
EndFunc

MsgBox(64, "Flag Generator", "Computing flag....")
Sleep(50000)
Local $flag_text = "zeropts<"
Sleep(50000)
Local $flag = compute()
Sleep(100000)
MsgBox(64, "Flag Generator", $flag_text & $flag & ">")
MsgBox(64, "Flag Generator", "Bye Bye...")
```

另一个只执行`system('cmd')`。



伪代码抄写：

```python
anime.subproc.addr = addr
anime.subproc.sise = size
```



#### 4. 启动子进程

`task`启动两个子进程，启动时设置了`CREATE_SUSPENDED`，进程创建后处于挂起(suspended)的状态，在调用`ResumeThread`之前子进程都不会真正执行代码。

狡猾的出题人在源码里写的并不是`CREATE_SUSPENDED`，而是直接写了魔数`4`。

```c
$runtime_newgoroutine("", "C:\Windows\System32\" & getanimepath(), 0, 0, 0, 4, 0, 0, $tstartup, $tprocess)
```

#### 5. 替换子进程

两次`signal4`，依次将两个子进程的`pid`和`tid`、子进程的父进程的`pid`和`tid`（这里子进程的父进程就是`task`）、[第3步](#3-查找资源地址)中查询到的`subproc.exe`的地址`address`和大小`size`、以及最重要的`data`发送给`anime`，初次两个子进程的`data`固定为`0`和`3`，之后的`data`如何确定会在之后说明。

`anime`会通过调用`ReadProcessMemory`从`task`读取加密的`subprocess.exe`，解密后映射到子进程的内存空间（解密过程的抄写见[上方](#3-查找资源地址)），实现对进程主模块的替换。

最后是对重要的`data`的处理。`anime`会把`data`保存下来，用于后面创建二叉树。然后在`data`中加入`flag`信息后写入子进程的`PEB->BeingDebugged`（出题人的恶意，调试器的反反调试插件会把`PEB->BeingDebugged`覆盖为`0`）。

为了避免混淆，把保存在`anime`中的`data`称为`data_anime`，处理后保存在`subProcess`中的称为`data_subproc`。处理过程和易懂的抄写如下。

![image-20220323132833257](/images/flag-checker.assets/image-20220323132833257.png)

其中`depth`是子进程到`task`进程的深度。显然，`data`和`data_anime`的低两位是完全等价的。



伪代码抄写：

```python
data_anime   = 0b{data.byte1,                              data.byte0} = data
data_subproc = 0b{data.byte1, anime.input[-1].byte(depth), data.byte0}

# anime.data_map[subproc.pid] = data_anime
subproc.data_anime = data_anime
# subproc.peb.BeingDebugged = data_subproc
subproc.data_subproc = data_subproc
```

#### 6. Resume

`task`调用`ResumeThread`，不过此时子进程主模块已经被替换成`subproc.exe`了。

`task`等待两个子进程退出。

#### 6.5 `subproc`里的假flag

`subproc`首先会向`stdout`写假flag。`subproc`并没有`terminal`，正常运行的时候并不会看到这个假flag。

![image-20220323140154454](/images/flag-checker.assets/image-20220323140154454.png)

<center><div style="border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">ねこptsCTF</div></center>

#### 7. 子进程深度判断

`signal2`: `subproc`发送`signal2`，`anime`会判断该进程与`task`进程之间的父子关系深度是否大于8。

如果深度已经达到8，则`subproc`会跳过下面的启动子进程过程，直接跳到[第9步](#9-树的构造)

```python
if subproc.depth > 8:
    goto step_9
```

#### 8. 子进程的子进程

`subproc`再来一次步骤[3](#3. 查找资源地址)、[4](#4. 启动子进程)、[5](#5. 替换子进程)、[6](#6. Resume)，区别在于行为主体从`task`变成了`subproc`，而且[第5步](#5-替换子进程)中的`data`不再是固定的`0`和`3`，而是由以下逻辑决定：

![image-20220323142542889](/images/flag-checker.assets/image-20220323142542889.png)

显然这段代码是个`switch`结构，四个主要分支的最明显的区别是`esi`和`r8d`的值不同（忽略其他几个寄存器，动态调试调起来的话就明白其他寄存器的两个取值无关紧要），没错，`esi`和`r8d`就是`subproc`的两个子进程的`data`。

```python
switch subproc.data_subproc >> 1:
    case 0:
    case 3:
        data0 = 0
        data1 = 3
    case 1:
    case 2:
        data0 = 2
        data1 = 1
```

现在已经知道`data`，那么`subproc`的两个子进程`subsubproc`对应的`data_anime`和`data_subproc`是多少呢，请自行推算。



于是每个子进程都会启动两个孙进程，直到进程深度达到8为止，于是该题实现了在程序父子关系上模拟二叉树。

#### 9. 树的构造

当子进程查询到自己的深度达到8，或者子进程的两个子进程都退出时，`subproc`会发送`signal6`，然后退出。

`anime`接收到`signal6`后，会将该进程到`task`进程父子关系间的所有进程插入二叉树。如果`data_anime`低位为0，则插入左子树，否则右子树。

![image-20220323144809719](/images/flag-checker.assets/image-20220323144809719.png)

```python
while curr_proc.pid != task.pid:
    # curr_proc.data_anime = anime.data_map[curr_pid]
    if curr_proc.data_anime % 2:
        bin_tree[curr_proc.parent.pid].rchild = curr_proc
    else:
        bin_tree[curr_proc.parent.pid].lchild = curr_proc
    curr_proc = curr_porc.parent
```

#### 10. 树的check

待到`task`的两个子进程都退出，也就是说所有的`subproc`都退出，`task`发送`signal7`，`anime`重要开始对树进行check。

首先`anime`会对树进行遍历：

![image-20220323150623694](/images/flag-checker.assets/image-20220323150623694.png)

显然，<span style="background-color:#000; color:#000">我看不出来这是什么遍历，构造数据测试后发现</span>这段是树的层级遍历。遍历时将节点的`data_anime`处理后转换为字符追加到字符串`from_user_input`末尾，易知这里的计算是取`data_anime`的低第二位，因为只有 1 byte，所以转换后的字符串完全由`0`和`1`组成。

最后丢弃字符串`from_user_input`的首位，也就是辈份最高的进程`task`对应的节点。

然后`anime`读取`task`内存空间中的一个图片资源，然后进行一系列无需深究的解码后得到数组`from_png_data`。

![image-20220323152337119](/images/flag-checker.assets/image-20220323152337119.png)



最后终于到了激动人心的 check 环节：

![image-20220323152826026](/images/flag-checker.assets/image-20220323152826026.png)

`sub_7FF78DFF4A20`在干什么并不重要（检查是否为素数），这段代码在筛选`from_png_data`中的元素（注意此处的idx是全局变量），取其第位与`from_user_input`的元素异或`'0'`后进行比较。最后取决于比较是否都相同，`anime`会向`task`返回一个`int`值，相同则是`0`；不同则是`rand() % 1336 +1`，总是是个非0值。



```python
from_user_input = ''
bin_tree.levelTraversal(proc => { from_user_input += str(proc.data_anime.byte1) })
global idx
result = True
for x in from_user_input:
    while not is_prime(idx // 3):
        idx += 3
    result &= (x^'0') == (from_png_data[idx]&1)
```

#### 11. continue

复杂吗，这样复杂的过程还要再来 109 遍，`task`从 2 开始，向`anime`传送用户输入字符串的第 2 位，展开第二个二叉树......

`task`会把每次`anime`在[第10步](#10-树的check)返回回来的`result`累加，最终结果是 0 则校验正确。



### 校验逻辑的理解与逆

了解了程序整体流程后，我们发现解题的关键在于`from_user_input`，也就是说关键在于`data_anime.byte1`以及树的结构。下面我们回顾[第5步](#5-替换子进程)、[第8步](#8-子进程的子进程)、[第9步](#9-树的构造)和[第10步](#10-树的check)，首先把这三步伪代码都贴过来:

```python
# step 5
data_anime   = 0b{data.byte1,                              data.byte0} = data
data_subproc = 0b{data.byte1, anime.input[-1].byte(depth), data.byte0}

# anime.data_map[subproc.pid] = data_anime
subproc.data_anime = data_anime
# subproc.peb.BeingDebugged = data_subproc
subproc.data_subproc = data_subproc

# step 8
switch subproc.data_subproc >> 1:
    case 0:
    case 3:
        data0 = 0
        data1 = 3
    case 1:
    case 2:
        data0 = 2
        data1 = 1

# step 9
while curr_proc.pid != task.pid:
    # curr_proc.data_anime = anime.data_map[curr_pid]
    if curr_proc.data_anime % 2:
        bin_tree[curr_proc.parent.pid].rchild = curr_proc
    else:
        bin_tree[curr_proc.parent.pid].lchild = curr_proc
    curr_proc = curr_porc.parent

# step 10
from_user_input = ''
bin_tree.levelTraversal(proc => { from_user_input += str(proc.data_anime.byte1) })
...
```

[第8步](#8-子进程的子进程)似乎还能优化：

```python
# step 8
d = subproc.data_subproc
if d.byte1 == d.byte2:
    data0 = 0
    data1 = 3
else:
    data0 = 2
    data1 = 1
```

加上[第5步](#5-替换子进程)的信息，我们知道`data_subproc.byte1`就是`anime.input[-1].byte(depth)`，于是

```python
d = subproc.data_subproc
if d.byte2 == anime.input[-1].byte(depth):
    data0 = 0
    data1 = 3
else:
    data0 = 2
    data1 = 1
```

加上[第9步](#9-树的构造)的信息，我们知道`data_anime.byte1`决定了节点是左子树或右子树：

```python
d = subproc.data_subproc
c0 = anime.input[-1].byte(depth)
c1 = anime.input[-1].byte(depth+1)
if d.byte2 == c0:
    lchild.data_subproc = 0b{0, c1, 0}
    rchild.data_subproc = 0b{1, c1, 1}
else:
    lchild.data_subproc = 0b{1, c1, 0}
    rchild.data_subproc = 0b{0, c1, 1}
```

消去[第10步](#10-树的check)中的`data_anime`：

```python
from_user_input = ''
bin_tree.levelTraversal(proc => { from_user_input += str(proc.data_subproc.byte2) })
```

综上，最后整理两个子节点最后对应的`from_user_data`子串：

```python
c0 = anime.input[-1].byte(depth)
if parent.data.byte2 == anime.input[-1].byte(depth):
    from_user_data_substr = '01'
else:
    from_user_data_substr = '10'
```

不得了，把他的逻辑逆过来就是：

```python
if from_user_data_substr_childs == '01':
    anime.input[-1].byte(depth) = int(from_user_data_substr_parent)
else:
    anime.input[-1].byte(depth) = int(from_user_data_substr_parent) ? 0 : 1
```

## Solve.py

```python
with open('./MEM_00000254F356D000_022E3000.mem', 'rb') as f:
    data = f.read()


def is_prime(a1):
    v3 = 0

    if (a1 == 2):
        v3 = 1
    elif (a1 >= 2 and (a1 & 1) != 0):
        for i in range(3, int(a1**0.5) + 1, 2):
            # ( i = 3i64; i * i <= a1; i += 2i64 )
            # {
            if not (a1 % i):
                # {
                v3 = 0
                return v3 & 1
        # }
        # }
        v3 = 1

    else:
        # {
        v3 = 0
    # }
    return v3 & 1
# }

idx = 0
def gen_png_data():
    global idx
    r = []
    i = 0
    while i < 1022:
        if is_prime(idx // 3):
            x = data[idx] % 2
            i += 1
            r.append(x)
        idx += 3
    return r


def lchild(i):
    return 2**(i + 1) - 2


def rchild(i):
    return 2**(i + 1) - 1


def selfi(i):
    return 2**(i) - 2


def solve(l: list):
    r = []
    for i in range(1, 8):
        lt = l[lchild(i)]
        rt = l[rchild(i)]
        s = l[selfi(i)]
        if lt == 1:
            b = 0 if s else 1
        else:
            b = 1 if s else 0
        r.append(b)
    return r[::-1]


if __name__ == '__main__':
    for i in range(110):
        d = gen_png_data()
        c = solve(d)
        c = int(''.join(map(str, c)), 2)
        print(chr(c), end='')

```

其中`MEM_00000254F356D000_022E3000.mem`是 dump 下来的`from_png_data`

运行脚本就可以得到`flag`：


```
zer0pts{me$s4g3_p4$$1ng_thru|p1pez_4r3_gr347!83157e16e25aa73f8a1fa1af06ac897a0f47d5be53ebb601953a43b6b3b4ca49}
```

## 总结

这题主要的难度在于调试。`task`使用`autoit`编写，这个语言并没有设计调试器，只能用`MsgBox`实现断点；`anime.exe`是无调试符号的`C++`程序，一些对象的类型和操作靠动态调试观察比较方便，而且用户代码中的所有`Winapi`都使用`CallObfuscator`包装，静态分析时看不到调用了什么`api`；`subproc.exe`是`go`语言编写，我个人感觉`go`语言动态调试更容易理解。

然而本题目严重依赖进程关系，可以调试器直接启动的只有`task.exe`，后续进程只能靠`attach`。调试`task`和`anime`的通信需要 4个调试器（两个用来调试`subproc`），最后构造树、观察树的遍历时更是用到了 6个调试器。

最后吐槽一下这题的二次元浓度：二次元浓度太少了，看到`findbestanimefrommyanimelist`，我本以为后续会出现什么奇怪的日本动画片，结果没有；我本以为`flag`里会有点二次元要素，结果没有；我以为[第10步](#10-树的check)中使用到的图片该有点二次元要素，结果是个风景照片。二次元浓度太少了！
