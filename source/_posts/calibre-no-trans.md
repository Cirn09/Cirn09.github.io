---
title: 别翻译我路径了，Calibre
date: 2025-02-05 16:11:36
tags:
  - 逆向
  - Calibre
typora-root-url: ..
typora-copy-images-to: ../images/${filename}/
---

 记录下人类和 Calibre 自动翻译战斗的伟大历史

 <!--more-->

## 背景

用过 Calibre 的同学都知道，Calibre 会在保存文件时将路径做一次拉丁化翻译。

从程序的角度来看是合理的，书库就是数据库，数据库底层数据保存方式应该尽可能通用，用户和书库交互的界面应当只有 Calibre，而不应关注底层实现。

问题是 Calibre 卡在了一个不上不下的位置，往下是路径完全 hash 化、往上是路径直接用书名。而且 Calibre 的翻译方式比较蠢，中文是拼音化、日语是罗马音化，日语中的汉字会被当成中文翻译成拼音，多音字会翻错，同音不同字翻译后结果一样（默认设置下，“好自为之.epub” 和 “耗子尾汁.epub” 会被当成同一本书，无法同时添加）。既不满足 hash 的单射，也不具备不翻译的语义。

由于上述种种问题，饱受折磨的人们做出了各种修改尝试。最正规的方法当然是给作者提意见，但作者阿三 Kovid Goyal 有点犟，不仅不接受这条建议，连别人写好的 patch 也拒绝合入（论坛讨论、issue、PR 链接就不找了，有兴趣的自己搜搜）。

正向的路走不通，只能走逆向的了。实现方案有两条路线：补丁方案和插件方案。

## 补丁方案

Calibre 是 Python 写的，Python 编译为 pyc 后打包。修改 Calibre 逻辑很简单：修改源码，编译新的 pyc，重新打包。修改源码和编译很简单，复杂的是重打包。针对 Calibre 官方包打包方式变更，补丁方案迭代了三版，就叫它们 v0、v1 和 v2 吧。

### 源码修改

实现书库不翻译要修改 `backend.py`，将这个文件中导入的 `ascii_filename` 函数修改为我们自定义的函数。`ascii_filename` 函数实现的就是拉丁化翻译，将它替换为一个普通的 `lambda x: x` 就实现了不翻译（当然还要处理路径非法字符）（在 [duli](https://github.com/kilesduli) 提示下发现 Calibre 自带了一个只处理非法字符、不做拉丁化的函数 `sanitize_file_name`）

### V0 - Zip 打包

早期 Calibre 使用 pyinstaller 的逻辑，将所有可能用到的 Python 包编译后 zip 打包，补丁修改时直接替换 zip 包里的文件即可。

V0 时期代表性的仓库有 [Snomiao](https://github.com/snomiao) 的 [calibre-utf8-path](https://github.com/snomiao/calibre-utf8-path)，这个仓库里也充分总结了前人的工作，对我后续开发帮忙挺大的。（我记得还有个更早的仓库，大约 700+ star，但现在搜 “Calibre 路径 翻译” 找到的几乎都是我的项目了，就不细找了）

### V1 - bypy 打包，Append patch

#### bypy 分析

首先感谢 [xinghui](https://github.com/xinghui233) 在 [calibre-utf8-path/6](https://github.com/snomiao/calibre-utf8-path/issues/6) 中对 Calibre bypy 打包的原始分析，帮忙很大。如果当时没看到他的初步分析的话，我可能根本提不起兴趣去做进一步的分析。

Calibre 5.1.0 开始使用作者自己搞的 [bypy](https://github.com/kovidgoyal/bypy) 打包，这个打包方式怎么说呢，可以说没什么必要。

打包的逻辑主要在 [bypy/freeze/__init__.py#L276](https://github.com/kovidgoyal/bypy/blob/4971a33c6a2ad71228e62699b15bd550bdab9b9a/bypy/freeze/__init__.py#L276)

```python
def freeze_python(
    base, dest_dir, include_dir, extensions_map, develop_mode_env_var='',
    path_to_user_env_vars='', remove_pyc_files=False
):
    files = collect_files_for_internment(base)
    frozen_file = os.path.join(dest_dir, 'python-lib.bypy.frozen')
    index_data = {}
    with open(frozen_file, 'wb') as frozen_file:
        for name, path in files.items():
            raw = open(path, 'rb').read()
            if name.lower().endswith('.pyc'):
                # remove the 16 byte magic tag at the start of pyc files used
                # for invalidation, since we dont do invalidation at all
                raw = raw[16:]
            index_data[name] = frozen_file.tell(), len(raw)
            frozen_file.write(raw)
    # from pprint import pprint
    # pprint(index_data)
    if len(index_data) > 9999:
        raise ValueError(
            'Too many files in python-lib have to switch'
            ' hash function to IntSaltHash and change C'
            ' template accordingly.')
    perfect_hash, code_to_get_index = get_c_code(index_data)
    values = []
    for k in sorted(index_data.keys(), key=perfect_hash):
        v = index_data[k]
        values.append(f'{{ {v[0]}u, {v[1]}u }}')
    vals = ','.join(values)
    tree = '\n'.join(bin_to_c(as_tree(index_data.keys(), extensions_map)))
    header = code_to_get_index + f'''
static void
get_value_for_hash_index(int index, unsigned long *offset, unsigned long *size)
{{
    typedef struct {{ unsigned long offset, size; }} Item;
    static const Item values[{len(values)}] = {{ {vals} }};
    if (index >= 0 && index < {len(values)}) {{
       *offset = values[index].offset; *size = values[index].size;
    }} else {{
    *offset = 0; *size = 0;
    }}
}}
static const unsigned char filesystem_tree[] = {{ {tree} }};
''' + importer_src_to_header(develop_mode_env_var, path_to_user_env_vars)
    with open(os.path.join(include_dir, 'bypy-data-index.h'), 'w') as f:
        f.write(header + '\n')
    if remove_pyc_files:
        remove_pyc_files_in(base)
        delete_empty_folders(base)
```

5-16 行做的是把所有 `pyc` 切掉开头 16 字节，首尾合并写入 `python-lib.bypy.frozen` 文件（以下简称 `bypy` 文件）。注意代码第 15 行，在拼接时还把 `pyc_name: (offset, pyc_size)` 保存下来了，后面会用到。另外为什么要切掉开头 16 字节呢，看 `pyc` 文件格式定义（此处省略文件格式定义数百字），可以看出前 16 字节在已知 Python 版本的前提下是没有用的。

剩下的代码用于生成 `bypy-data-index.h`，这个文件最后会参与 Calibre 的编译。这个文件的关键是 `typedef struct { unsigned long offset, size; } Item;` 和 `static const unsigned char filesystem_tree[] = {{ {tree} }};`，分别保存了 `pyc` 在 `bypy` 文件中的偏移、长度，以及名字。重要的是它们的顺序是完全保持一致的（文件名字母序）。至于 hash？不关心。

总结一下，原先的 `zip` 打包是把文件和文件索引都保存在了 `zip` 文件中。而新的 `bypy` 打包，将文件保存在了 `bypy` 文件中，将索引保存在了 Calibre 二进制中。优点是索引速度可能有这么一丁点提升，进而提升一丁点启动速度吧，毕竟可以使用 hash 查表了。缺点是压缩率降低了，相比 `zip` 真压缩格式，`bypy` 打包对每个 `pyc` 节省 16 字节空间。没什么意义，不如找个通用打包格式。

#### #### 临时方案

在上面的 [calibre-utf8-path/6](https://github.com/snomiao/calibre-utf8-path/issues/6) 中，由[Hsuan](https://hsuan9522.medium.com/%E6%9A%B4%E5%8A%9B%E7%A0%B4%E8%A7%A3-calibre-%E4%B8%AD%E6%96%87%E7%9B%AE%E9%8C%84-2b124f92f54) 提出了一种新的临时方案，并由 [Kuriko Moe](https://github.com/kurikomoe) 给出了开源仓库 [calibre-utf8-path](https://github.com/kurikomoe/calibre-utf8-path)。这个方案利用了 `CALIBRE_DEVELOP_FROM`，让 Calibre 强制加载本地代码。这个模式原版是方便调试用的，从讨论来看并不完美：

>> 感谢您的修改，现在保存为中文目录，方便多了，简直造福人类！就是有个小问题：在 calibre 主目录，点击阅读图书的时候，epub 等格式的图书没法直接点开，需要打开所在目录，然后点击该文件才能阅读， 稍微有点不方便！不知道是否有解决办法？

> 实际上是 dev 模式下，viewer 的源代码需要被 jit 编译后使用，Linux 上能直接编译后启动，Windows 可能缺乏 python 环境导致编译失败。
> 在我的机子上大概要 10s 才能显示界面。我倒是可以改成直接显示 viewer 的模式，你可以测试一下。（在 windows 虚拟机里似乎没问题）

#### V1 - Append patch

最后看一下被我称为 V1 版本的 append patch 方案。

思考一下 `bypy` 方案能否二次修改呢，显然是可以的。只要找到 `backend.pyc` 在 `bypy` 中的偏移量，覆盖上去应该就可以了。

##### 找 `pyc` 偏移

怎么找偏移呢？得从打包过程入手，首先找到 `filesystem_tree` 中 `backend` 字符串（ `calibre/db/backend.pyc`）的位置，然后找到指向它的指针，然后找到 `filesystem_tree`头部指针，计算出 `backend` 在数组中的索引。然后找到 `struct { unsigned long offset, size; }` 数组起始地址，加上索引值就能得到 `backend` 的文件偏移和文件大小了。

找字符串不困难，直接搜就完事了：![image-20250205183039595](/images/calibre-no-trans/image-20250205183039595.png)

指针也不难找，把字符串的文件偏移换算成 VA，继续搜就完事了![image-20250205183213041](/images/calibre-no-trans/image-20250205183213041.png)

那么指针数组首地址在哪呢？基于这个数组按文件字母序排序，直接找第一个文件 `Crypto/Cipher/AES.pyc` 的指针地址就可以了（实际上为了避免后续引入新的更靠前的文件，自动 patch 代码里会自动按这个模式找到最靠前的 `*.pyc` 字符串指针）

![image-20250205183325470](/images/calibre-no-trans/image-20250205183325470.png)

下一个问题是 `{ unsigned long offset, size; }` 数组怎么找。这也不难，注意到首个文件的 `offset` 必定为 0，第二个文件的 `offset` 必定等于首个文件的 `size`，后续每个文件的 `offset` 都等于上一个文件的 `offset + size`。按照这个模式在二进制中搜索即可。

![image-20250205184216350](/images/calibre-no-trans/image-20250205184216350.png)

太好了，那我们就这么做吧。

等等，有聪明的小伙伴已经发现了，我们新的 `backend.pyc` 大小不能比原版 `backend.pyc`大，因为 `backend.pyc` 在 `bypy` 文件中间，前后都有其他文件，能使用的空间已经限制死了。

原版 `backend` 是直接 `import` 拉丁化函数，而我们的新函数是另外写的，编译出来大小一定比原版大。通过优化编译以缩小 `pyc` 可以吗？不行，因为 Calibre 编译时已经是最优编译了。

想来可行的方法就只有把二进制中 `{ unsigned long offset, size; }` 也修改了。把新 `backend` 放在 `bypy` 最后面，然后修改二进制，让 offset 指向新的位置，size 也改成新的。

完美，这套逻辑无懈可击，最重要的是 patch 逻辑都可以写成脚本，全自动完成。配合 GitHub Actions，每天检查更新，有更新就自动 patch、自动 release。

### V1.1 - shrink patch

V1 自动跑了一段时间后，平静的生活被 [1 号 issue](https://github.com/Cirn09/calibre-do-not-translate-my-path/issues/1)（Mac覆盖原文件后程序崩溃）打破。原来是 Mac 的二进制开了代码签名，V1 patch 修改了二进制中的数据（offset 和 size），代码执行到这里会触发签名校验失败。

好在 Mac 编译流程中，`bypy` 文件被认为是资源文件，对它并没有签名校验。解决这个问题的关键就是缩减 patch 后的 `backend` 大小。仔细观察 `backend.py`，注意到代码中有很多多行 SQL 语句，太好了，压缩一下这些语句就能省下一大堆空间。足够给新增的函数用了。

![image-20250205192234770](/images/calibre-no-trans/image-20250205192234770.png)

### V1.2 - update

过了一段时间后，感觉每次更新太麻烦了，每次更新都是要先更新 Calibre 本体，然后下载补丁，替换文件。每次要下载两次文件，安装两次。V1.2 干脆把检查更新链接也 patch 了，指向本项目版本号；更新链接指向本项目 release 链接。每次 patch 时顺便把安装包重新打包起来，这样每次更新只要下载、安装一次就可以了。

替换后的 URL 缩短了，所以不用从别的地方扣出空间了。

### V2 - Mac `-Os`

GitHub Actions 传来噩耗，Calibre v7.17 Mac 版本 patch 失败。

看了一下是阿三把 Mac 的编译选项改了 [304192c](https://github.com/kovidgoyal/calibre/commit/304192c10c43a0fc105085a4b99b8c2dd985b095)，编译选项变成了 '-Os'，这个选项会优先优化产物大小，这导致 macOS 产物编译出来的二进制模式发生了变化。打包方式并没有变，但是编译改变导致之前用于识别的模式不用继续用了。

变化的是指向字符串的指针数组，从直接保存指针，改成了保存当前地址和字符串之间的偏移量。这确实优化了大小，原先保存指针需要 8 字节，现在保存偏移量只需要 4 字节就够了。大约节约了 7844*4=31376。

![image-20250205201017506](/images/calibre-no-trans/image-20250205201017506.png)

好在其他平台的编译选项没变，只改 Mac 的逻辑即可。

## V3 - Plugin 方案

终于到了 Plugin 方案，这个方案是我在思考如何实现“发送到设备”路径不翻译时想到的，这个方案的实现方式大概是众多前人和我都忽视了的盲点……

先说下 Patch 方案实现“发送到设备”路径不翻译的难点。Calibre 对不同协议的发送逻辑做出了不同的实现，这些地方写法各有区别，patch 方案很难保证跨版本自动 patch 不出问题（实际也不是问题，这部分代码已经好几年没动过了）。另外这些源码中很难抠出空间了，没有 SQL 语句这种稍微去掉点空格就能优化出大片空间的软柿子了……

### 原理

思考这样两个 Python 文件：

``` Python
# a.py
from pprint import pprint
while True:
    pprint(1)
    
# b.py
import a
a.pprint = lambda x: print(9)
```

在同一进程空间内，`b.py` 执行前，`a.py` 会不停输出 `1`；而 `b.py` 执行后，`a.py` 中的 `pprint` 被替换成了我们自己的匿名函数，行为就变成了不停输出 `9`。这就是动态语言的灵活之处了，hook 只需要简单赋值即可。

理解了这个原理之后，NoTrans 插件就不难理解了，`import` 对应模块，赋值替换对应实现即可。真的是朴实无华且枯燥……
