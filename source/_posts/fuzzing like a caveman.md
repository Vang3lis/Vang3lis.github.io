---
title: 'Fuzzing Like A Caveman 译'
date: 2022-04-28 21:00:21
category: Fuzz
tags: [fuzz]
published: true
hideInList: false
feature: 
isTop: false
---

原文为 [Fuzzing-Like-A-Caveman](https://h0mbre.github.io/Fuzzing-Like-A-Caveman)，本文章仅为个人理解的翻译和备份

## Intoduction

在过去的几个月里，我一直在被动地消化大量与模糊测试相关的材料，因为我主要尝试将我的`Windows exploitation game`从`Noob-level`提高到`1%-Less-Noob-Level`，而且我发现它相当的迷人，在这篇文章中，我将向你们展示如何创建一个非常简单变异的`fuzzer`，希望我们可以利用这个`fuzzer`在一些开源项目中找到一些`crashes`。

我们将创建的这个`fuzzer`是跟随着 [@gynvael’s fuzzing tutorial on YouTube](https://twitter.com/gynvael?ref_src=twsrc%5Egoogle%7Ctwcamp%5Eserp%7Ctwgr%5Eauthor)实现的，我不知道`Gynvael`有视频，所以现在我有几十个小时的内容可以添加到无止尽的`watch/read`列表中。

我也必须要说[Brandon Faulk’s fuzzing streams](https://www.youtube.com/user/gamozolabs/videos)是非常好的，虽然我不能理解近乎`99%`的`Brandon says`所说的事情，但是这些视频的内容是吸引人的，到目前为止，我个人最喜欢的是他对`calc.exe`和`c-tags`的`fuzzing`，他也有一个关于`fuzzing`概念特别好的介绍的视频：[NYU Fuzzing Talk](https://www.youtube.com/watch?v=SngK4W4tVc0)

## Picking a Target

我想找一个用`C`或者`C++`写的从文件中解析数据的`binary`，我最先找到的其中一个就是从`images`中解析`Exif`数据的`binaries`，我们也想选一个几乎没有安全隐患的目标，因为我们将会实时的公布这些发现。

从[exif](https://www.media.mit.edu/pia/Research/deepview/exif.html)可知，基本上，`Exif`文件格式是跟`JPEG`文件格式是相同的，`Exif`插入一些`image/digicam`信息数据和缩略图 (`thumbnail image`) 到`JPEG`中以符合`JPEG`的规范，因此你可以通过`JEPG`兼容的浏览器/图片查看器/图片修饰软件等查看`Exif`格式的文件，就像通常查看`JPEG`文件一样。

因此，`Exif`可将元数据类型插入到符合`JPEG`规范的文件中，并且市面上存在不少有助于解析这些数据的`programs/utilities`。

## Getting Started

我们将用`Python3`去构建一个初级的变异的`fuzzer`，巧妙地（或者不那么巧妙地）改变合规的`Exif-filled`的`JPEGs`，并将它们喂给解析器希望能得到`crash`，我们还将开发`x86 Kali Linux`发行版。

首先，我们需要一个合规的`Exif-filled`的`JPEG`，谷歌搜素`Sample JPEG with Exif`就有助于我们找到这个[repo](https://github.com/ianare/exif-samples/tree/master/jpg)，我将会用`Canon_40D.jpg`用于测试。

## Getting to Know the JPEG and EXIF Spec

在我们将`Python`写入`Sublime Text`之前，让我们首先花一点来了解`JPEG`和`Exif`规范，一遍我们可以避免一些更明显的损坏图像，以至于解析器不会试图解析它以浪费宝贵的`fuzzing cycles`。

从前文提到的[规范概述](https://www.media.mit.edu/pia/Research/deepview/exif.html)可知所有的`JPEG`图片都是以`0xFFD8`开头，以`0xFFD9`结尾，这前几个字节就是所谓的`magic bytes`，在`*Nix system`中通过`magic bytes`可以直接识别出文件类型。

```
root@kali:~# file Canon_40D.jpg 
Canon_40D.jpg: JPEG image data, JFIF standard 1.01, resolution (DPI), density 72x72, segment length 16, Exif Standard: [TIFF image data, little-endian, direntries=11, manufacturer=Canon, model=Canon EOS 40D, orientation=upper-left, xresolution=166, yresolution=174, resolutionunit=2, software=GIMP 2.4.5, datetime=2008:07:31 10:38:11, GPS-Data], baseline, precision 8, 100x68, components 3
```

我们可以去掉`.jpg`并获得相同的输出

```
root@kali:~# file Canon
Canon: JPEG image data, JFIF standard 1.01, resolution (DPI), density 72x72, segment length 16, Exif Standard: [TIFF image data, little-endian, direntries=11, manufacturer=Canon, model=Canon EOS 40D, orientation=upper-left, xresolution=166, yresolution=174, resolutionunit=2, software=GIMP 2.4.5, datetime=2008:07:31 10:38:11, GPS-Data], baseline, precision 8, 100x68, components 3
```

如果我们 `hexdump` 这个图片，我们可以看到最初的几个字节和最后的几个字节实际上就是`0xFFD8`和`0xFFD9`

```
root@kali:~# hexdump Canon
0000000 d8ff e0ff 1000 464a 4649 0100 0101 4800
------SNIP------
0001f10 5aed 5158 d9ff 
```

在规范概述中另外一个有趣的信息是`markers`是以`0xFF`开头的，有几种一直的静态标记 (`marders`) 例如：
+ the 'Start of Image'(SOI) marker: 0xFFD8
+ APP1 marker: 0xFFE1
+ generic markers: 0xFFXX
+ the 'End of Image'(EOI) marker: 0xFFD9

因为我们不像去改变`image`的长度或者文件的类型，所以让我们继续并计划尽可能保持`SOI`和`EOI`标记完整，我们并不想插入`0xFFD9`到图像中间，以至于这会截断图像或者导致解析器以`non-crashy way`而产生混乱（`Non-crashy`并不是一个真实的词），这也可能会被误导也许我们也应该在字节流中随机放置`EOI`标记？来我们试试看。

## Starting Our Fuzzer

这我们将做的第一件事情就是提取`JPEG`中的所有字节，我们希望将其用作我们的`valid`输入蓝本，当然我们会对其进行变异。

我们的代码将会从写成这样开始：

```python
#!/usr/bin/env python3

import sys

# read bytes from our valid JPEG and return them in a mutable bytearray 
def get_bytes(filename):
    f = open(filename, "rb").read()
    return bytearray(f)

if len(sys.argv) < 2:
    print("Usage: JPEGfuzz.py <valid_jpg>")
else:
    filename = sys.argv[1]
    data = get_bytes(filename)
```

如果我们希望看到这个数据是长什么样的，我们可以在数组中打印最初10个左右的字节值，然后看看我们将会如何与其进行交互，我们将仅是临时添加一些类似如下的东西：

```python
else:
    filename = sys.argv[1]
    data = get_bytes(filename)
    counter = 0
    for x in data:
        if counter < 10:
            print(x)
        counter += 1
```

运行这个，显示我们正将其转换为整齐的十进制整数，在我看来，这让一切变得更为容易。

```
root@kali:~# python3 fuzzer.py Canon_40D.jpg 
255
216
255
224
0
16
74
70
73
70
```

让我们快速地看看是否可以从我们的字节数组中创建一个新的合规的 `JPEG`，我们将把这个函数添加到我们的代码中并运行它。

```python
def create_new(data):
    f = open("mutated.jpg", "wb+")
    f.write(data)
    f.close()
```

所以现在在我们的字典中可以获得一个`mutated.jpg`，让我们`hash`比较这两个文件，看它们是否匹配的上.

```
root@kali:~# shasum Canon_40D.jpg mutated.jpg 
c3d98686223ad69ea29c811aaab35d343ff1ae9e  Canon_40D.jpg
c3d98686223ad69ea29c811aaab35d343ff1ae9e  mutated.jpg
```

有趣，我们可以有两个相同的文件，现在我们可以在创建`mutated.jpg`之前变异数据。

## Mutating

我们将保持我们的 `fuzzer` 相对简单，并且只是实现两种不同的变异方式，这些方法如下：

+ 位翻转（`bit flipping`）
+ 用`Gynvael's 'Magic Numbers'`覆盖字节序列（`overwriting byte sequences with Gynvael’s ‘Magic Numbers’`）

让我们开始位翻转吧，`255`（`0xFF`）在二进制中将是`11111111`，如果我们随机翻转这个数字中的一位，假设在`index`数字为`2`，我们将得到`11011111`，这个新数字会是`223`（`0xDF`）。

我不能完全确认这种变异方式和随机选择一个`0-255`的数字并且重写一个新的数字有着什么区别，我的直觉说位翻转与用任意字节随机覆盖字节非常相似。

让我们继续，假设我们想去仅翻转`1%`的比特，我们在`Python`中通过如下代码得到这个数字。

```
num_of_flips = int((len(data) - 4) * .01)
```

我们想从我们的字节数组的长度中减去4，因为我们不想计算在我们的数组中的最初的两个字节或最后连个字节，因为这些是`SOI`和`EOI`标记，我们意在保持原样。

接下来我们将想要随机选择许多的`indexes`然后将这些`indexes`作为位翻转的目标，我们将继续创建一组可以修改的可能的`indexes`，然后选择其中的`num_of_flips`个进行随机的位翻转。

```python
indexes = range(4, (len(data) - 4))

chosen_indexes = []

# iterate selecting indexes until we've hit our num_of_flips number
counter = 0
while counter < num_of_flips:
    chosen_indexes.append(random.choice(indexes))
    counter += 1
```

让我们把`import random`加到我们的`script`中，并添加这些`debug`的`print`状态以确保所有的一切都能成功运转。

```python
print("Number of indexes chosen: " + str(len(chosen_indexes)))
print("Indexes chosen: " + str(chosen_indexes))
```

我们的`function`现在是像这样：

```python
def bit_flip(data):
    num_of_flips = int((len(data) - 4) * .01)
    indexes = range(4, (len(data) - 4))

    chosen_indexes = []

    # iterate selecting indexes until we've hit our num_of_flips number
    counter = 0
    while counter < num_of_flips:
        chosen_indexes.append(random.choice(indexes))
        counter += 1

    print("Number of indexes chosen: " + str(len(chosen_indexes)))
    print("Indexes chosen: " + str(chosen_indexes))
```

如果我们跑这个脚本，我们将会得到一个预期的不错的输出。

```
root@kali:~# python3 fuzzer.py Canon_40D.jpg 
Number of indexes chosen: 79
Indexes chosen: [6580, 930, 6849, 6007, 5020, 33, 474, 4051, 7722, 5393, 3540, 54, 5290, 2106, 2544, 1786, 5969, 5211, 2256, 510, 7147, 3370, 625, 5845, 2082, 2451, 7500, 3672, 2736, 2462, 5395, 7942, 2392, 1201, 3274, 7629, 5119, 1977, 2986, 7590, 1633, 4598, 1834, 445, 481, 7823, 7708, 6840, 1596, 5212, 4277, 3894, 2860, 2912, 6755, 3557, 3535, 3745, 1780, 252, 6128, 7187, 500, 1051, 4372, 5138, 3305, 872, 6258, 2136, 3486, 5600, 651, 1624, 4368, 7076, 1802, 2335, 3553]
```

接下来我们需要实际变异这些`indexes`上的字节，我们需要位翻转他们，我选择用一个非常`hacky`的方式去做这件事，你尽可以随意的实现你自己的方法，我们准备转换这些`indexes`上的字节到二进制字符串，并将其补齐为8位数长，让我们添加这个代码，看看我们在说些什么，我们将会转换这字节的值（就是转为十进制）到二进制串，并如果它少于8位数长时在其前面填充0，这最后一行就是调试时的临时打印。

```python
for x in chosen_indexes:
    current = data[x]
    current = (bin(current).replace("0b",""))
    current = "0" * (8 - len(current)) + current
```

正如你所示，我们有一个不错的二进制数字作为字符串的输出。

```
root@kali:~# python3 fuzzer.py Canon_40D.jpg 
10100110
10111110
10010010
00110000
01110001
00110101
00110010
-----SNIP-----
```

现在对于其中的每一个，我们将会随机挑选一个`index`然后翻转它，例如，`10100110`，如果选择`index 0`，我们得到是`1`，我们将会翻转其成为`0`。

这段代码段最后的考虑是这些是字符串而不是整数，所以我们需要做的最后一件事情就是将翻转的二进制字符串转换为整数。

我们将创建一个空白的列表、把每个数字降入这个列表、翻转我们随机挑选的数字，然后从所有的列表成员中构造一个新的字符串（我们必须使用这个中间列表的步骤，因为字符串是可变异的），最后我们将其转换为整数，然后返回数据给我们的`create_new()`函数以创建一个新的`JPEG`。

我们的全部脚本目前是像这样：

```python
#!/usr/bin/env python3

import sys
import random

# read bytes from our valid JPEG and return them in a mutable bytearray 
def get_bytes(filename):
    f = open(filename, "rb").read()
    return bytearray(f)

def bit_flip(data):
    num_of_flips = int((len(data) - 4) * .01)
    indexes = range(4, (len(data) - 4))

    chosen_indexes = []

    # iterate selecting indexes until we've hit our num_of_flips number
    counter = 0
    while counter < num_of_flips:
        chosen_indexes.append(random.choice(indexes))
        counter += 1

    for x in chosen_indexes:
        current = data[x]
        current = (bin(current).replace("0b",""))
        current = "0" * (8 - len(current)) + current
        
        indexes = range(0,8)

        picked_index = random.choice(indexes)

        new_number = []

        # our new_number list now has all the digits, example: ['1', '0', '1', '0', '1', '0', '1', '0']
        for i in current:
            new_number.append(i)

        # if the number at our randomly selected index is a 1, make it a 0, and vice versa
        if new_number[picked_index] == "1":
            new_number[picked_index] = "0"
        else:
            new_number[picked_index] = "1"

        # create our new binary string of our bit-flipped number
        current = ''
        for i in new_number:
            current += i

        # convert that string to an integer
        current = int(current,2)

        # change the number in our byte array to our new number we just constructed
        data[x] = current

    return data


# create new jpg with mutated data
def create_new(data):
    f = open("mutated.jpg", "wb+")
    f.write(data)
    f.close()

if len(sys.argv) < 2:
    print("Usage: JPEGfuzz.py <valid_jpg>")

else:
    filename = sys.argv[1]
    data = get_bytes(filename)
    mutated_data = bit_flip(data)
    create_new(mutated_data)
```

## Analyzing Mutation

如果我们跑我们的脚本，我们可以利用`shasum`这个输出，然后与原本的`JPEG`进行比较。

```
root@kali:~# shasum Canon_40D.jpg mutated.jpg 
c3d98686223ad69ea29c811aaab35d343ff1ae9e  Canon_40D.jpg
a7b619028af3d8e5ac106a697b06efcde0649249  mutated.jpg
```

这个看起来很有戏，因为它们现在有着不同的哈希值，我们未来通过用一个程序（被称为[Beyond Compare](https://www.scootersoftware.com/)或`bcompare`）比较它们以进行分析，我们将会有着不同高亮的两个`hexdumps`

![bcompare.png](/img/fuzzing_like_a_caveman/bcompare.png)

正如你所是，在这一个屏幕的共享中，我们有着三个字节的不同，它们已经进行了它们的位翻转，这个初始的在左侧，而变异的样本在右侧。

这个变异的方法似乎起作用了，让我们继续实现我们第二个变异的方法。

## Gynvael’s Magic Numbers

在前文提到的`GynvaelColdwind`的[Basics of fuzzing’ stream](https://www.youtube.com/watch?v=BrDujogxYSk&t=2545)，他枚举了一些在程序上可以有着毁灭影响的`magic numbers`，通常，这些数字和数据类型大小和算数引起的错误有关，这些讨论的数字是：

+ 0xFF
+ 0x7F
+ 0x00
+ 0xFFFF
+ 0x0000
+ 0xFFFFFFFF
+ 0x00000000
+ 0x80000000 <—- minimum 32-bit int
+ 0x40000000 <—- just half of that amount
+ 0x7FFFFFFF <—- max 32-bit int

如果在`malloc()`或其他类型的操作过程中对这些类型的值执行任何类型的算术运算，溢出可能就是很常见，例如如果你加`0x1`到`0xFF`在一个字节的寄存器上，它将转到`0x00`，这是无意的行为，`HEVD`实际上已经有一个与这个理念类似的整数溢出`bug`。

假设我们的`fuzzer`选择`0x7FFFFFFF`作为它想要使用的魔数，该值是4个字节长，所以我们必须在数组中找到一个字节索引，并覆盖该字节加上接下来的三个字节。让我们继续并开始在我们的`fuzzer`中实现它。

## Implementing Mutation Method #2

首先我们将想要创建一个类似`Gynvael`做的一个元组列表，在这元组中的第一个数字是`magic numbers`的字节数，第二个数字是十进制上的字节的值。

```python
def magic(data):
    magic_vals = [
    (1, 255),
    (1, 255),
    (1, 127),
    (1, 0),
    (2, 255),
    (2, 0),
    (4, 255),
    (4, 0),
    (4, 128),
    (4, 64),
    (4, 127)
    ]

    picked_magic = random.choice(magic_vals)

    print(picked_magic)
```

如果我们跑这个脚本，我门就可以看到它随机在取一个`magic`值元组

```python3
root@kali:~# python3 fuzzer.py Canon_40D.jpg 
(4, 64)
root@kali:~# python3 fuzzer.py Canon_40D.jpg 
(4, 128)
root@kali:~# python3 fuzzer.py Canon_40D.jpg 
(4, 0)
root@kali:~# python3 fuzzer.py Canon_40D.jpg 
(2, 255)
root@kali:~# python3 fuzzer.py Canon_40D.jpg 
(4, 0)
```

我们现在需要用新的`magic`1到4字节的值复写到`JPEG`中的1到4字节的值上，我们将要建立我们可能的`indexes`就像之前那个方法一样，随机选择`index`然后覆盖这个`index`上的字节用我们`picked_magic`数字。

所以，如果例如我们取的是`(4, 128)`，我们知道这是4个字节，所以这个魔数是`0x80000000`，所以我们将做些类似如下的事情：

```python
byte[x] = 128
byte[x+1] = 0
byte[x+2] = 0
byte[x+3] = 0
```

总而言之，我们的函数如下所示：

```python
def magic(data):
    magic_vals = [
    (1, 255),
    (1, 255),
    (1, 127),
    (1, 0),
    (2, 255),
    (2, 0),
    (4, 255),
    (4, 0),
    (4, 128),
    (4, 64),
    (4, 127)
    ]

    picked_magic = random.choice(magic_vals)

    length = len(data) - 8
    index = range(0, length)
    picked_index = random.choice(index)

    # here we are hardcoding all the byte overwrites for all of the tuples that begin (1, )
    if picked_magic[0] == 1:
        if picked_magic[1] == 255:			# 0xFF
            data[picked_index] = 255
        elif picked_magic[1] == 127:			# 0x7F
            data[picked_index] = 127
        elif picked_magic[1] == 0:			# 0x00
            data[picked_index] = 0

    # here we are hardcoding all the byte overwrites for all of the tuples that begin (2, )
    elif picked_magic[0] == 2:
        if picked_magic[1] == 255:			# 0xFFFF
            data[picked_index] = 255
            data[picked_index + 1] = 255
        elif picked_magic[1] == 0:			# 0x0000
            data[picked_index] = 0
            data[picked_index + 1] = 0

    # here we are hardcoding all of the byte overwrites for all of the tuples that being (4, )
    elif picked_magic[0] == 4:
        if picked_magic[1] == 255:			# 0xFFFFFFFF
            data[picked_index] = 255
            data[picked_index + 1] = 255
            data[picked_index + 2] = 255
            data[picked_index + 3] = 255
        elif picked_magic[1] == 0:			# 0x00000000
            data[picked_index] = 0
            data[picked_index + 1] = 0
            data[picked_index + 2] = 0
            data[picked_index + 3] = 0
        elif picked_magic[1] == 128:			# 0x80000000
            data[picked_index] = 128
            data[picked_index + 1] = 0
            data[picked_index + 2] = 0
            data[picked_index + 3] = 0
        elif picked_magic[1] == 64:			# 0x40000000
            data[picked_index] = 64
            data[picked_index + 1] = 0
            data[picked_index + 2] = 0
            data[picked_index + 3] = 0
        elif picked_magic[1] == 127:			# 0x7FFFFFFF
            data[picked_index] = 127
            data[picked_index + 1] = 255
            data[picked_index + 2] = 255
            data[picked_index + 3] = 255
        
    return data
```

## Analyzing Mutation #2

现在我们运行脚本，然后在`Beyond Compare`中分析结果，我们可以看到这个两个字节的`0xA6 0x76`将被复写成`0xFF 0xFF`。

![bcompare2.png](/img/fuzzing_like_a_caveman/bcompare2.png)

这正是我们想实现的。

## Starting to Fuzz

现在我们有了两个可靠的编译数据的方法，我们需要做：

+ 用我们的函数的其中一个变异数据
+ 用变异的数据创建一个新的图片
+ 把变异的图片喂给我们的二进制用于解析
+ 捕获任何`Segmentation faults`并且`log`产生这个的图片

### Victim?

对于我们的受害者程序，我们将用`site:github.com "exif" language:c`搜索`Google`，以找到用 C 编写的引用了`exif`的`Github`项目。

快速浏览将我们带到`https://github.com/mkttanabe/exif`

我们可以通过`git cloning`这个仓库和用`README`中的`building with gcc`指令来安装该程序（我已经把编译好的二进制文件放到了`/usr/bin`，仅仅是为了方便）

让我们先来看看这个程序怎么处理我们合规的`JPEG`吧。

```
root@kali:~# exif Canon_40D.jpg -verbose
system: little-endian
  data: little-endian
[Canon_40D.jpg] createIfdTableArray: result=5

{0TH IFD} tags=11
tag[00] 0x010F Make
        type=2 count=6 val=[Canon]
tag[01] 0x0110 Model
        type=2 count=14 val=[Canon EOS 40D]
tag[02] 0x0112 Orientation
        type=3 count=1 val=1 
tag[03] 0x011A XResolution
        type=5 count=1 val=72/1 
tag[04] 0x011B YResolution
        type=5 count=1 val=72/1 
tag[05] 0x0128 ResolutionUnit
        type=3 count=1 val=2 
tag[06] 0x0131 Software
        type=2 count=11 val=[GIMP 2.4.5]
tag[07] 0x0132 DateTime
        type=2 count=20 val=[2008:07:31 10:38:11]
tag[08] 0x0213 YCbCrPositioning
        type=3 count=1 val=2 
tag[09] 0x8769 ExifIFDPointer
        type=4 count=1 val=214 
tag[10] 0x8825 GPSInfoIFDPointer
        type=4 count=1 val=978 

{EXIF IFD} tags=30
tag[00] 0x829A ExposureTime
        type=5 count=1 val=1/160 
tag[01] 0x829D FNumber
        type=5 count=1 val=71/10 
tag[02] 0x8822 ExposureProgram
        type=3 count=1 val=1 
tag[03] 0x8827 PhotographicSensitivity
        type=3 count=1 val=100 
tag[04] 0x9000 ExifVersion
        type=7 count=4 val=0 2 2 1 
tag[05] 0x9003 DateTimeOriginal
        type=2 count=20 val=[2008:05:30 15:56:01]
tag[06] 0x9004 DateTimeDigitized
        type=2 count=20 val=[2008:05:30 15:56:01]
tag[07] 0x9101 ComponentsConfiguration
        type=7 count=4 val=0x01 0x02 0x03 0x00 
tag[08] 0x9201 ShutterSpeedValue
        type=10 count=1 val=483328/65536 
tag[09] 0x9202 ApertureValue
        type=5 count=1 val=368640/65536 
tag[10] 0x9204 ExposureBiasValue
        type=10 count=1 val=0/1 
tag[11] 0x9207 MeteringMode
        type=3 count=1 val=5 
tag[12] 0x9209 Flash
        type=3 count=1 val=9 
tag[13] 0x920A FocalLength
        type=5 count=1 val=135/1 
tag[14] 0x9286 UserComment
        type=7 count=264 val=0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 (omitted)
tag[15] 0x9290 SubSecTime
        type=2 count=3 val=[00]
tag[16] 0x9291 SubSecTimeOriginal
        type=2 count=3 val=[00]
tag[17] 0x9292 SubSecTimeDigitized
        type=2 count=3 val=[00]
tag[18] 0xA000 FlashPixVersion
        type=7 count=4 val=0 1 0 0 
tag[19] 0xA001 ColorSpace
        type=3 count=1 val=1 
tag[20] 0xA002 PixelXDimension
        type=4 count=1 val=100 
tag[21] 0xA003 PixelYDimension
        type=4 count=1 val=68 
tag[22] 0xA005 InteroperabilityIFDPointer
        type=4 count=1 val=948 
tag[23] 0xA20E FocalPlaneXResolution
        type=5 count=1 val=3888000/876 
tag[24] 0xA20F FocalPlaneYResolution
        type=5 count=1 val=2592000/583 
tag[25] 0xA210 FocalPlaneResolutionUnit
        type=3 count=1 val=2 
tag[26] 0xA401 CustomRendered
        type=3 count=1 val=0 
tag[27] 0xA402 ExposureMode
        type=3 count=1 val=1 
tag[28] 0xA403 WhiteBalance
        type=3 count=1 val=0 
tag[29] 0xA406 SceneCaptureType
        type=3 count=1 val=0 

{Interoperability IFD} tags=2
tag[00] 0x0001 InteroperabilityIndex
        type=2 count=4 val=[R98]
tag[01] 0x0002 InteroperabilityVersion
        type=7 count=4 val=0 1 0 0 

{GPS IFD} tags=1
tag[00] 0x0000 GPSVersionID
        type=1 count=4 val=2 2 0 0 

{1ST IFD} tags=6
tag[00] 0x0103 Compression
        type=3 count=1 val=6 
tag[01] 0x011A XResolution
        type=5 count=1 val=72/1 
tag[02] 0x011B YResolution
        type=5 count=1 val=72/1 
tag[03] 0x0128 ResolutionUnit
        type=3 count=1 val=2 
tag[04] 0x0201 JPEGInterchangeFormat
        type=4 count=1 val=1090 
tag[05] 0x0202 JPEGInterchangeFormatLength
        type=4 count=1 val=1378 

0th IFD : Model = [Canon EOS 40D]
Exif IFD : DateTimeOriginal = [2008:05:30 15:56:01]
```

我们可以看到这个程序解析出`tags`并说明与它们相关联的字节值，这正是我们想要找到的。

### Chasing Segfaults

理想情况下，我们将会喂给这个程序一些变异的数据，然后得到`segfault`，意味着我们已找到了`bug`。我遇到的问题是当我为了`Segmentation fault`消息监控`stdout`和`sterr`是，它从未出现，这是因为`Segmentation fault`消息是来自我们的命令行`shell`而不是二进制，这意味着`shell`收到一个`SIGSEGV`信号，并且会反馈打印这个信息。

我发现监控这个的有一种方法用`pexpect Python module`的`use()`方法和`pipes Python module`的`quote()`方法。

我们将加一个新的函数，这个将输入`counter`作为参数，这个参数为我们`fuzzer`迭代的轮数，另外一个参数是变异的数据，如果我们看到`Segmentation`在我们`run()`指令的输出里面，我们将把这个变异的数据写入文件并保存，这样我们就可以有让二进制`crash`的`JPEG`图片了。

让我们创建一个新的称为`crashes`文件夹，然后我们将在这里面保存`JPEGs`，这些导致`crashes`的图片将会以`crash.<fuzzing iteration (counter)>.jpg`的格式保存，所以如果这个`fuzzing`迭代了100次导致了`a crash`，我们应该会得到这样的文件：`/crashes/crash.100.jpg`

我们将持续以保持每100次模糊测试迭代的计数打印到终端的同一行，我们的函数如下所示：

```python
def exif(counter,data):
    command = "exif mutated.jpg -verbose"

    out, returncode = run("sh -c " + quote(command), withexitstatus=1)

    if b"Segmentation" in out:
        f = open("crashes/crash.{}.jpg".format(str(counter)), "ab+")
        f.write(data)

    if counter % 100 == 0:
        print(counter, end="\r")
```

接下来，我们将在我们代码的最下面替换我们执行的`stub`以跑一个计数器，一旦我们达到了`1000`次迭代，我们就停止`fuzzing`，我们也将让我们的`fuzzer`随机选择一个我们变异的方法，所以它或使用位翻转或使用魔数，让我们运行它，然后在结束时检测我们的`crashes`文件夹。

当`fuzzer`结束后，你可以看到我们获得了大概`30`个`crashes`。

```
root@kali:~/crashes# ls
crash.102.jpg  crash.317.jpg  crash.52.jpg   crash.620.jpg  crash.856.jpg
crash.129.jpg  crash.324.jpg  crash.551.jpg  crash.694.jpg  crash.861.jpg
crash.152.jpg  crash.327.jpg  crash.559.jpg  crash.718.jpg  crash.86.jpg
crash.196.jpg  crash.362.jpg  crash.581.jpg  crash.775.jpg  crash.984.jpg
crash.252.jpg  crash.395.jpg  crash.590.jpg  crash.785.jpg  crash.985.jpg
crash.285.jpg  crash.44.jpg   crash.610.jpg  crash.84.jpg   crash.987.jpg
```

我们现在可以用一行快速的代码证实这个结果：`root@kali:~/crashes# for i in *.jpg; do exif "$i" -verbose > /dev/null 2>&1; done` ，记住我们可以将`STDOUT`和`STDERR`都导向`/dev/null`，因为`Segmentation fault`是来自`shell`，而不是来自二进制文件。

我们跑上面这指令，以下为输出：

```
root@kali:~/crashes# for i in *.jpg; do exif "$i" -verbose > /dev/null 2>&1; done
Segmentation fault
Segmentation fault
Segmentation fault
Segmentation fault
Segmentation fault
Segmentation fault
Segmentation fault
-----SNIP-----
```

你不可以看到它们的所有，但是这里确实有30个`segfaults`，所以所有的事情都似乎按照计划的进行。

## Triaging Crashes

现在我们有大约30个`crashes`和导致`crashes`的`JPEGs`，下一步是分析这些`crahes`并找出其中它们有多少是`unique`，在这里我们可以用一些我们看`Brandon Faulk`的视频学到的东西，用`Beyond Compare`快速看一遍`crash`的样本，我们发现大部分都是因为我们的`bit_flip()`变异而不是`magic()`变异方法产生的，这很有趣，作为测试，当我们运行的过程中，我们可以关闭函数选择的随机性，只使用`magic()`变异方式运行100000次迭代，再看看我们是否会遇到任何`crahes`

## Using ASan to Analyze Crashes

`ASan`是`Address Sanitizer`，它是较新版本的`gcc`附带的有效工具，允许用户在编译二进制文件使用`-fsanitize=address`参数打开，并在发生内存访问错误时（即使是那些导致`crash`的时候）获得非常详细的信息，很显然，我们这里已经预选择了会崩溃的输入，所以我们错过该有效工具（笔者认为这里指的意思是在`fuzzing`的时候没有用`ASan`），但也许我们将会保存它在其他时间（使用）（笔者认为这里指的是在调`crash`的时候使用带`ASan`的编译后的二进制文件）。

为了使用 ASan，我跟着[the Fuzzing Project](https://fuzzing-project.org/tutorial2.html)的思路，然后用如下`flags: cc -fsanitize=address -ggdb -o exifsan sample_main.c exif.c`重新编译了`exif` 

然后为了方便使用我将`exifsan` 移动到`/usr/bin`，如果我们在`crash`样本上运行这个新编译的二进制文件，让我们看看输出吧

```
root@kali:~/crashes# exifsan crash.252.jpg -verbose
system: little-endian
  data: little-endian
=================================================================
==18831==ERROR: AddressSanitizer: heap-buffer-overflow on address 0xb4d00758 at pc 0x00415b9e bp 0xbf8c91f8 sp 0xbf8c91ec
READ of size 4 at 0xb4d00758 thread T0                                                                                              
    #0 0x415b9d in parseIFD /root/exif/exif.c:2356
    #1 0x408f10 in createIfdTableArray /root/exif/exif.c:271
    #2 0x4076ba in main /root/exif/sample_main.c:63
    #3 0xb77d0ef0 in __libc_start_main ../csu/libc-start.c:308
    #4 0x407310 in _start (/usr/bin/exifsan+0x2310)

0xb4d00758 is located 0 bytes to the right of 8-byte region [0xb4d00750,0xb4d00758)
allocated by thread T0 here:                                                                                                        
    #0 0xb7aa2097 in __interceptor_malloc (/lib/i386-linux-gnu/libasan.so.5+0x10c097)
    #1 0x415a9f in parseIFD /root/exif/exif.c:2348
    #2 0x408f10 in createIfdTableArray /root/exif/exif.c:271
    #3 0x4076ba in main /root/exif/sample_main.c:63
    #4 0xb77d0ef0 in __libc_start_main ../csu/libc-start.c:308

SUMMARY: AddressSanitizer: heap-buffer-overflow /root/exif/exif.c:2356 in parseIFD
Shadow bytes around the buggy address:
  0x369a0090: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x369a00a0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x369a00b0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x369a00c0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x369a00d0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
=>0x369a00e0: fa fa fa fa fa fa fa fa fa fa 00[fa]fa fa 04 fa
  0x369a00f0: fa fa 00 06 fa fa 06 fa fa fa fa fa fa fa fa fa
  0x369a0100: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x369a0110: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x369a0120: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x369a0130: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
  Shadow gap:              cc
==18831==ABORTING
```

很不错，我们不仅可以获得详细信息，而且`ASan`还为我们区分了错误类别，告诉我们`crash`地址并提供一个很好的堆栈调用，正如你所见，我们在`exif.c`的`parseIFD `函数中执行了一个4字节的读操作。

```
READ of size 4 at 0xb4d00758 thread T0                                                                                              
    #0 0x415b9d in parseIFD /root/exif/exif.c:2356
    #1 0x408f10 in createIfdTableArray /root/exif/exif.c:271
    #2 0x4076ba in main /root/exif/sample_main.c:63
    #3 0xb77d0ef0 in __libc_start_main ../csu/libc-start.c:308
    #4 0x407310 in _start (/usr/bin/exifsan+0x2310)
```

由于现在这都是标准二进制输出，我们实际上可以对这些`crash`进行分类并尝试理解它们，让我们首先尝试对`crash`进行冗余删除，有可能我们所有的30次崩溃都是同一个错误。也有可能我们有30次`unqiue crashes`（不太可能哈哈），所以我们需要解决这个问题。

让我们再次调用`Python`脚本，我们将遍历该文件夹，对每次`crash`运行启用`ASan`的二进制文件，并记录每个`crash`地址的位置，我们还将尝试捕获它是`READ`还是`WRITE`操作，例如，对于`crash.252.jpg`，我们将日志文件名字格式化为：`crash.252.HBO.b4f00758.READ`，并将`ASan`输出写入日志，这样我们甚至在打开日志之前就知道导致它的`crash`图片名、`bug`的类别、地址和操作。 （我会在最后贴检测和分类的脚本，太恶心了，我讨厌它）

在我们的`crashes`文件夹上运行检测和分类脚本后，我们现在可以看到我们已经分类了我们的`crashes`并且有一些非常有趣的东西。

```
crash.102.HBO.b4f006d4.READ
crash.102.jpg
crash.129.HBO.b4f005dc.READ
crash.129.jpg
crash.152.HBO.b4f005dc.READ
crash.152.jpg
crash.317.HBO.b4f005b4.WRITE
crash.317.jpg
crash.285.SEGV.00000000.READ
crash.285.jpg
------SNIP-----
```

在那里进行了一次大`SNIP`之后，在我的30次`crash`中，我只有一次`WRITE`操作。你无法从截断的输出中看到，但我也有很多`SEGV`错误因为`NULL`地址被引用了(0x00000000)

让我们检查一下我们修改后的`fuzzer`，它只运行了100000次迭代的`magic()`编译，看看它是否找到任何`bugs`

```
root@kali:~/crashes2# ls
crash.10354.jpg  crash.2104.jpg   crash.3368.jpg   crash.45581.jpg  crash.64750.jpg  crash.77850.jpg  crash.86367.jpg  crash.94036.jpg
crash.12771.jpg  crash.21126.jpg  crash.35852.jpg  crash.46757.jpg  crash.64987.jpg  crash.78452.jpg  crash.86560.jpg  crash.9435.jpg
crash.13341.jpg  crash.23547.jpg  crash.39494.jpg  crash.46809.jpg  crash.66340.jpg  crash.78860.jpg  crash.88799.jpg  crash.94770.jpg
crash.14060.jpg  crash.24492.jpg  crash.40953.jpg  crash.49520.jpg  crash.6637.jpg   crash.79019.jpg  crash.89072.jpg  crash.95438.jpg
crash.14905.jpg  crash.25070.jpg  crash.41505.jpg  crash.50723.jpg  crash.66389.jpg  crash.79824.jpg  crash.89738.jpg  crash.95525.jpg
crash.18188.jpg  crash.27783.jpg  crash.41700.jpg  crash.52051.jpg  crash.6718.jpg   crash.81206.jpg  crash.90506.jpg  crash.96746.jpg
crash.18350.jpg  crash.2990.jpg   crash.43509.jpg  crash.54074.jpg  crash.68527.jpg  crash.8126.jpg   crash.90648.jpg  crash.98727.jpg
crash.19441.jpg  crash.30599.jpg  crash.43765.jpg  crash.55183.jpg  crash.6987.jpg   crash.82472.jpg  crash.90745.jpg  crash.9969.jpg
crash.19581.jpg  crash.31243.jpg  crash.43813.jpg  crash.5857.jpg   crash.70713.jpg  crash.83282.jpg  crash.92426.jpg
crash.19907.jpg  crash.31563.jpg  crash.44974.jpg  crash.59625.jpg  crash.77590.jpg  crash.83284.jpg  crash.92775.jpg
crash.2010.jpg   crash.32642.jpg  crash.4554.jpg   crash.64255.jpg  crash.77787.jpg  crash.84766.jpg  crash.92906.jpg
```

这有一堆`crashes`

## Getting Serious, Conclusion

`fuzzer`可以进行很多优化，目前它真的很粗糙，只是为了演示非常基本的编译`fuzzing`，`bug`分类过程也是一团糟，感觉整个过程都很`hacky`，我想我需要看更多`@gamozolabs`的视频，也许下次我们进行模糊测试时，我们会尝试一个更难的目标，用`Rust`或`Go`等很酷的语言编写模糊测试，我们将会尝试真正改进分类过程/利用其中一个`bugs`！

感谢博文中提到的每个人，非常感谢

直到下一次！

## Code

`JPEGfuzz.py`

```python
#!/usr/bin/env python3

import sys
import random
from pexpect import run
from pipes import quote

# read bytes from our valid JPEG and return them in a mutable bytearray 
def get_bytes(filename):

    f = open(filename, "rb").read()

    return bytearray(f)

def bit_flip(data):

    num_of_flips = int((len(data) - 4) * .01)

    indexes = range(4, (len(data) - 4))

    chosen_indexes = []

    # iterate selecting indexes until we've hit our num_of_flips number
    counter = 0
    while counter < num_of_flips:
        chosen_indexes.append(random.choice(indexes))
        counter += 1

    for x in chosen_indexes:
        current = data[x]
        current = (bin(current).replace("0b",""))
        current = "0" * (8 - len(current)) + current
        
        indexes = range(0,8)

        picked_index = random.choice(indexes)

        new_number = []

        # our new_number list now has all the digits, example: ['1', '0', '1', '0', '1', '0', '1', '0']
        for i in current:
            new_number.append(i)

        # if the number at our randomly selected index is a 1, make it a 0, and vice versa
        if new_number[picked_index] == "1":
            new_number[picked_index] = "0"
        else:
            new_number[picked_index] = "1"

        # create our new binary string of our bit-flipped number
        current = ''
        for i in new_number:
            current += i

        # convert that string to an integer
        current = int(current,2)

        # change the number in our byte array to our new number we just constructed
        data[x] = current

    return data

def magic(data):

    magic_vals = [
    (1, 255),
    (1, 255),
    (1, 127),
    (1, 0),
    (2, 255),
    (2, 0),
    (4, 255),
    (4, 0),
    (4, 128),
    (4, 64),
    (4, 127)
    ]

    picked_magic = random.choice(magic_vals)

    length = len(data) - 8
    index = range(0, length)
    picked_index = random.choice(index)

    # here we are hardcoding all the byte overwrites for all of the tuples that begin (1, )
    if picked_magic[0] == 1:
        if picked_magic[1] == 255:			# 0xFF
            data[picked_index] = 255
        elif picked_magic[1] == 127:		# 0x7F
            data[picked_index] = 127
        elif picked_magic[1] == 0:			# 0x00
            data[picked_index] = 0

    # here we are hardcoding all the byte overwrites for all of the tuples that begin (2, )
    elif picked_magic[0] == 2:
        if picked_magic[1] == 255:			# 0xFFFF
            data[picked_index] = 255
            data[picked_index + 1] = 255
        elif picked_magic[1] == 0:			# 0x0000
            data[picked_index] = 0
            data[picked_index + 1] = 0

    # here we are hardcoding all of the byte overwrites for all of the tuples that being (4, )
    elif picked_magic[0] == 4:
        if picked_magic[1] == 255:			# 0xFFFFFFFF
            data[picked_index] = 255
            data[picked_index + 1] = 255
            data[picked_index + 2] = 255
            data[picked_index + 3] = 255
        elif picked_magic[1] == 0:			# 0x00000000
            data[picked_index] = 0
            data[picked_index + 1] = 0
            data[picked_index + 2] = 0
            data[picked_index + 3] = 0
        elif picked_magic[1] == 128:		# 0x80000000
            data[picked_index] = 128
            data[picked_index + 1] = 0
            data[picked_index + 2] = 0
            data[picked_index + 3] = 0
        elif picked_magic[1] == 64:			# 0x40000000
            data[picked_index] = 64
            data[picked_index + 1] = 0
            data[picked_index + 2] = 0
            data[picked_index + 3] = 0
        elif picked_magic[1] == 127:		# 0x7FFFFFFF
            data[picked_index] = 127
            data[picked_index + 1] = 255
            data[picked_index + 2] = 255
            data[picked_index + 3] = 255
        
    return data

# create new jpg with mutated data
def create_new(data):

    f = open("mutated.jpg", "wb+")
    f.write(data)
    f.close()

def exif(counter,data):

    command = "exif mutated.jpg -verbose"

    out, returncode = run("sh -c " + quote(command), withexitstatus=1)

    if b"Segmentation" in out:
        f = open("crashes2/crash.{}.jpg".format(str(counter)), "ab+")
        f.write(data)

    if counter % 100 == 0:
        print(counter, end="\r")

if len(sys.argv) < 2:
    print("Usage: JPEGfuzz.py <valid_jpg>")

else:
    filename = sys.argv[1]
    counter = 0
    while counter < 100000:
        data = get_bytes(filename)
        functions = [0, 1]
        picked_function = random.choice(functions)
        if picked_function == 0:
            mutated = magic(data)
            create_new(mutated)
            exif(counter,mutated)
        else:
            mutated = bit_flip(data)
            create_new(mutated)
            exif(counter,mutated)

        counter += 1
```

`triage.py`

```python3
#!/usr/bin/env python3

import os
from os import listdir

def get_files():

    files = os.listdir("/root/crashes/")

    return files

def triage_files(files):

    for x in files:

        original_output = os.popen("exifsan " + x + " -verbose 2>&1").read()
        output = original_output
        
        # Getting crash reason
        crash = ''
        if "SEGV" in output:
            crash = "SEGV"
        elif "heap-buffer-overflow" in output:
            crash = "HBO"
        else:
            crash = "UNKNOWN"
        

        if crash == "HBO":
            output = output.split("\n")
            counter = 0
            while counter < len(output):
                if output[counter] == "=================================================================":
                    target_line = output[counter + 1]
                    target_line2 = output[counter + 2]
                    counter += 1
                else:
                    counter += 1
            target_line = target_line.split(" ")
            address = target_line[5].replace("0x","")
            

            target_line2 = target_line2.split(" ")
            operation = target_line2[0]
            

        elif crash == "SEGV":
            output = output.split("\n")
            counter = 0
            while counter < len(output):
                if output[counter] == "=================================================================":
                    target_line = output[counter + 1]
                    target_line2 = output[counter + 2]
                    counter += 1
                else:
                    counter += 1
            if "unknown address" in target_line:
                address = "00000000"
            else:
                address = None

            if "READ" in target_line2:
                operation = "READ"
            elif "WRITE" in target_line2:
                operation = "WRITE"
            else:
                operation = None

        log_name = (x.replace(".jpg","") + "." + crash + "." + address + "." + operation)
        f = open(log_name,"w+")
        f.write(original_output)
        f.close()



files = get_files()
triage_files(files)
```