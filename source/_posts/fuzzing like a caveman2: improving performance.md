---
title: 'Fuzzing Like A Caveman2: Improving Performance 译'
date: 2022-06-11 00:47:21
category: Fuzz
tags: [fuzz]
published: true
hideInList: false
feature: 
isTop: false
---

原文为 [Fuzzing Like A Caveman 2: Improving Performance](https://h0mbre.github.io/Fuzzing-Like-a-Caveman-2)，本文章仅为个人理解的翻译和备份

## Introduction

在`Fuzzing like a Caveman`一节中（讲述了编写`fuzzer`），我们将关注提升我们之前`fuzzer`的性能，这意味着这将不会有任何大规模的改变，我们仅关注提升我们之前那篇博客做的东西，这意味着在这篇博文结束时我们最终还是会带着一个非常基本的`fuzzer`（仅让它变得更快！！），希望在不同目标上能发现更多的`bugs`，在这篇文章中，我们不会真正修改多线程或多进程，我们会将其保存到后续的`fuzzing`文章中

我觉得我需要在这里加一个**免责声明**，我远不是一个专业的开发者，在这一点上，我只是没有足够的编程经验像一个更有经验的程序员那样来识别提高性能的机会，我将使用我粗略的技术和有限的编程知识来改进我们之前的`fuzzer`，嗯就是这样，这个生成的代码将不漂亮，不完美，但是它会比我上一篇的文章做的`fuzzer`更好，这需要提到，所有的测试都是在`VMWare Workstation`中的`1 CPU 1 Core`的`x86 Kali`虚拟机上

让我们花点时间在这篇博客中定义`better`，我这里所说的`better`是我们可以在n次`fuzzing`迭代中会更快迭代，就是这样，我们将花点时间完成重写`fuzzer`、用一个更酷的语言、选择一个更稳固的目标并在以后用更先进的模糊测试技术

**很明显，如果你还没阅读上一篇文章，你将会跟不上！**

## Analyzing Our Fuzzer

很明显，我们最后的`fuzzer`是起作用的！我们在我们的目标文件中发现了一些`bugs`，但是我们知道当我们完成`fuzzer`时，我们还遗漏了一些优化，让我们再看看上一篇文章中的`fuzzer`（为了测试做了一些小的改动）。

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
        print("Segfault!")

    #if counter % 100 == 0:
    #	print(counter, end="\r")

if len(sys.argv) < 2:
    print("Usage: JPEGfuzz.py <valid_jpg>")

else:
    filename = sys.argv[1]
    counter = 0
    while counter < 1000:
        data = get_bytes(filename)
        functions = [0, 1]
        picked_function = random.choice(functions)
        picked_function = 1
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

你或许注意到一些改变，我们改了：

+ 每100次迭代注释掉迭代计数器的打印语句
+ 添加用于提醒我们任何`Segfaults`的打印语句
+ 硬编码1k次迭代
+ 临时添加一行新代码`picked_function=1`，以便我们消除我们测试中的随机性，我们只一直用一种变异策略（`bit_flip()`）

让我们用一些分析工具跑我们新版的`fuzzer`，我们可以实际分析在我们程序执行时我们花费的时间

我们可以利用`cProfile`的`Python module`，看看在`1000`次`fuzzing`迭代中我们把时间花在哪里了，如果你还记得的，这个程序会把一个合规的`JPEG`文件作为路径参数，因此我们完整的命令行语法就像`python3 -m cProfile -s cumtime JPEGfuzzer.py ~/jpegs/Canon_40D.jpg`所示。

**需要注意的是，添加这个`cProfile`工具会降低性能，我测试时是没带这个工具，在该文章中对于这个迭代大小，这似乎是不会造成显著差异**

在这次运行之后，我们可以看到我们的程序的输出，并且我们可以看到在执行中我们最花时间的地方。

```
2476093 function calls (2474812 primitive calls) in 122.084 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
     33/1    0.000    0.000  122.084  122.084 {built-in method builtins.exec}
        1    0.108    0.108  122.084  122.084 blog.py:3(<module>)
     1000    0.090    0.000  118.622    0.119 blog.py:140(exif)
     1000    0.080    0.000  118.452    0.118 run.py:7(run)
     5432  103.761    0.019  103.761    0.019 {built-in method time.sleep}
     1000    0.028    0.000  100.923    0.101 pty_spawn.py:316(close)
     1000    0.025    0.000  100.816    0.101 ptyprocess.py:387(close)
     1000    0.061    0.000    9.949    0.010 pty_spawn.py:36(__init__)
     1000    0.074    0.000    9.764    0.010 pty_spawn.py:239(_spawn)
     1000    0.041    0.000    8.682    0.009 pty_spawn.py:312(_spawnpty)
     1000    0.266    0.000    8.641    0.009 ptyprocess.py:178(spawn)
     1000    0.011    0.000    7.491    0.007 spawnbase.py:240(expect)
     1000    0.036    0.000    7.479    0.007 spawnbase.py:343(expect_list)
     1000    0.128    0.000    7.409    0.007 expect.py:91(expect_loop)
     6432    6.473    0.001    6.473    0.001 {built-in method posix.read}
     5432    0.089    0.000    3.818    0.001 pty_spawn.py:415(read_nonblocking)
     7348    0.029    0.000    3.162    0.000 utils.py:130(select_ignore_interrupts)
     7348    3.127    0.000    3.127    0.000 {built-in method select.select}
     1000    0.790    0.001    1.777    0.002 blog.py:15(bit_flip)
     1000    0.015    0.000    1.311    0.001 blog.py:134(create_new)
     1000    0.100    0.000    1.101    0.001 pty.py:79(fork)
     1000    1.000    0.001    1.000    0.001 {built-in method posix.forkpty}
-----SNIP-----
```

对于这种类型的分析，我们并不真正关心我们有多少`segfaults`，因为我们并没有真正修改变异方法或比较不同的方法。当然这里会有一些随机性，因为`crash`需要额外的处理，但现在就可以了。

我只截了我们累计花费了超过`1.0s`的代码段，你可以看到迄今为止我们花费了最多的时间在`blog.py:140(exif)`，在`122s`中占了惊人的`118s`，我们`exif()`函数似乎成为我们性能的最大的问题。

我们可以看到在这个函数下我们花费的大部分时间都是直接跟这个函数相关的，从`pexpect`用法中我们可以看到大量对`pty`模块的诉求（`appeal to the pty module`，笔者感觉就是对该模块的调用），让我们重写我们的函数用`subprocess`模块的`Popen`，看看是否我们可以提升性能吧。

我们重定义`exif()`函数如下：

```python
def exif(counter,data):

    p = Popen(["exif", "mutated.jpg", "-verbose"], stdout=PIPE, stderr=PIPE)
    (out,err) = p.communicate()

    if p.returncode == -11:
        f = open("crashes2/crash.{}.jpg".format(str(counter)), "ab+")
        f.write(data)
        print("Segfault!")

    #if counter % 100 == 0:
    #	print(counter, end="\r")
```

我们的性能报告如下：

```
2065580 function calls (2065443 primitive calls) in 2.756 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
     15/1    0.000    0.000    2.756    2.756 {built-in method builtins.exec}
        1    0.038    0.038    2.756    2.756 subpro.py:3(<module>)
     1000    0.020    0.000    1.917    0.002 subpro.py:139(exif)
     1000    0.026    0.000    1.121    0.001 subprocess.py:681(__init__)
     1000    0.099    0.000    1.045    0.001 subprocess.py:1412(_execute_child)
 -----SNIP-----
```

区别是什么，这用重定义的`exif()`函数的`fuzzer`在相同数目的工作量下仅仅需要`2s`！！以前的`fuzzer: 122s`，新的`fuzzer: 2.7s`

## Improving Further in Python

让我们尝试在`Python`中继续改进我们的`fuzzer`,首先，让我们得到一个好的基准，可以让我们对照着执行，我们将让我们的优化的`Python fuzzer`迭代50000次，我们将再次使用`cProfile`模块来获得一些关于我们花费时间的细粒度统计。

```
102981395 function calls (102981258 primitive calls) in 141.488 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
     15/1    0.000    0.000  141.488  141.488 {built-in method builtins.exec}
        1    1.724    1.724  141.488  141.488 subpro.py:3(<module>)
    50000    0.992    0.000  102.588    0.002 subpro.py:139(exif)
    50000    1.248    0.000   61.562    0.001 subprocess.py:681(__init__)
    50000    5.034    0.000   57.826    0.001 subprocess.py:1412(_execute_child)
    50000    0.437    0.000   39.586    0.001 subprocess.py:920(communicate)
    50000    2.527    0.000   39.064    0.001 subprocess.py:1662(_communicate)
   208254   37.508    0.000   37.508    0.000 {built-in method posix.read}
   158238    0.577    0.000   28.809    0.000 selectors.py:402(select)
   158238   28.131    0.000   28.131    0.000 {method 'poll' of 'select.poll' objects}
    50000   11.784    0.000   25.819    0.001 subpro.py:14(bit_flip)
  7950000    3.666    0.000   10.431    0.000 random.py:256(choice)
    50000    8.421    0.000    8.421    0.000 {built-in method _posixsubprocess.fork_exec}
    50000    0.162    0.000    7.358    0.000 subpro.py:133(create_new)
  7950000    4.096    0.000    6.130    0.000 random.py:224(_randbelow)
   203090    5.016    0.000    5.016    0.000 {built-in method io.open}
    50000    4.211    0.000    4.211    0.000 {method 'close' of '_io.BufferedRandom' objects}
    50000    1.643    0.000    4.194    0.000 os.py:617(get_exec_path)
    50000    1.733    0.000    3.356    0.000 subpro.py:8(get_bytes)
 35866791    2.635    0.000    2.635    0.000 {method 'append' of 'list' objects}
   100000    0.070    0.000    1.960    0.000 subprocess.py:1014(wait)
   100000    0.252    0.000    1.902    0.000 selectors.py:351(register)
   100000    0.444    0.000    1.890    0.000 subprocess.py:1621(_wait)
   100000    0.675    0.000    1.583    0.000 selectors.py:234(register)
   350000    0.432    0.000    1.501    0.000 subprocess.py:1471(<genexpr>)
 12074141    1.434    0.000    1.434    0.000 {method 'getrandbits' of '_random.Random' objects}
    50000    0.059    0.000    1.358    0.000 subprocess.py:1608(_try_wait)
    50000    1.299    0.000    1.299    0.000 {built-in method posix.waitpid}
   100000    0.488    0.000    1.058    0.000 os.py:674(__getitem__)
   100000    1.017    0.000    1.017    0.000 {method 'close' of '_io.BufferedReader' objects}
-----SNIP-----
```

50000次迭代总共花费了我们141s，与我们正在处理的相比，这个性能是不错的了，我们之前做1000次迭代可用了122s！再次过滤只在我们花费了超过1.0s时间的地方，我们发现还是在`exif()`中花费了大部分时间，但是我们同样在`bit_flip()`中也发现了一些性能问题，因为我们在这累计花费了25s，让我们尝试优化一点这个函数吧。

让我们继续并贴出旧`bit_flip()`函数

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
```

诚然，这个函数有点笨拙，我们可以用更好的逻辑来大大简化它，我发现在我有限的编程经历中经常会出现下面这种情况，你尽可以用你想要的所有花哨的深奥的编程知识，但是如果你编程背后的逻辑是不健全的，那么程序的性能就会受到影响。

让我们减少我们所做的类型转换的数量，例如从`int`转成`str`或者反过来，让我们在编译器中用更少的代码，我们可以通过重新定义的`bit_flip()`完成我们想要的，如下所示：

```python
def bit_flip(data):

    length = len(data) - 4

    num_of_flips = int(length * .01)

    picked_indexes = []
    
    flip_array = [1,2,4,8,16,32,64,128]

    counter = 0
    while counter < num_of_flips:
        picked_indexes.append(random.choice(range(0,length)))
        counter += 1


    for x in picked_indexes:
        mask = random.choice(flip_array)
        data[x] = data[x] ^ mask

    return data
```

如果我们使用这个新的函数并监控结果，我们将获得以下的性能等级：

```
59376275 function calls (59376138 primitive calls) in 135.582 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
     15/1    0.000    0.000  135.582  135.582 {built-in method builtins.exec}
        1    1.940    1.940  135.582  135.582 subpro.py:3(<module>)
    50000    0.978    0.000  107.857    0.002 subpro.py:111(exif)
    50000    1.450    0.000   64.236    0.001 subprocess.py:681(__init__)
    50000    5.566    0.000   60.141    0.001 subprocess.py:1412(_execute_child)
    50000    0.534    0.000   42.259    0.001 subprocess.py:920(communicate)
    50000    2.827    0.000   41.637    0.001 subprocess.py:1662(_communicate)
   199549   38.249    0.000   38.249    0.000 {built-in method posix.read}
   149537    0.555    0.000   30.376    0.000 selectors.py:402(select)
   149537   29.722    0.000   29.722    0.000 {method 'poll' of 'select.poll' objects}
    50000    3.993    0.000   14.471    0.000 subpro.py:14(bit_flip)
  7950000    3.741    0.000   10.316    0.000 random.py:256(choice)
    50000    9.973    0.000    9.973    0.000 {built-in method _posixsubprocess.fork_exec}
    50000    0.163    0.000    7.034    0.000 subpro.py:105(create_new)
  7950000    3.987    0.000    5.952    0.000 random.py:224(_randbelow)
   202567    4.966    0.000    4.966    0.000 {built-in method io.open}
    50000    4.042    0.000    4.042    0.000 {method 'close' of '_io.BufferedRandom' objects}
    50000    1.539    0.000    3.828    0.000 os.py:617(get_exec_path)
    50000    1.843    0.000    3.607    0.000 subpro.py:8(get_bytes)
   100000    0.074    0.000    2.133    0.000 subprocess.py:1014(wait)
   100000    0.463    0.000    2.059    0.000 subprocess.py:1621(_wait)
   100000    0.274    0.000    2.046    0.000 selectors.py:351(register)
   100000    0.782    0.000    1.702    0.000 selectors.py:234(register)
    50000    0.055    0.000    1.507    0.000 subprocess.py:1608(_try_wait)
    50000    1.452    0.000    1.452    0.000 {built-in method posix.waitpid}
   350000    0.424    0.000    1.436    0.000 subprocess.py:1471(<genexpr>)
 12066317    1.339    0.000    1.339    0.000 {method 'getrandbits' of '_random.Random' objects}
   100000    0.466    0.000    1.048    0.000 os.py:674(__getitem__)
   100000    1.014    0.000    1.014    0.000 {method 'close' of '_io.BufferedReader' objects}
-----SNIP-----
```

从指标中可以看出，此时我们在`bit_flip()`中只花了14秒的累计时间！在我们最后一轮中，这个将近花费了25秒，在这一点上，这几乎是两倍的速度了，在我看来，我们在这里的优化工作做得很好。

现在我们有了我们理想的`Python`基准测试（请记住，这里可能会有多进程或多线程的机会，但我们把这个想法留在下次），让我们继续把我们的`fuzzer`移植到一个新的语言`c++`，并测试其性能。

## New Fuzzer in C++

首先，让我们继续运行我们新优化的`python fuzzer`，并将其运行`100k`次`fuzzing`迭代，看看需要多长时间。

`118749892 function calls (118749755 primitive calls) in 256.881 seconds`

100k次迭代仅需256s！这性能摧毁了我们之前写的`fuzzer`

这将是我们用`c++`实现尝试要打败的基准测试，现在，就像我对`Python`开发的细微点的不熟悉程度，将其乘十，你就会发现，我对`c++`不熟悉的程度了，这段代码或许会贻笑大方，但是这是我目前能做的最好的，我们可以对应我们之前的`Python`代码解释每个函数。

让我们逐个函数来看看，然后描述其实现

```c++
//
// this function simply creates a stream by opening a file in binary mode;
// finds the end of file, creates a string 'data', resizes data to be the same
// size as the file moves the file pointer back to the beginning of the file;
// reads the data from the into the data string;
//
std::string get_bytes(std::string filename)
{
    std::ifstream fin(filename, std::ios::binary);

    if (fin.is_open())
    {
        fin.seekg(0, std::ios::end);
        std::string data;
        data.resize(fin.tellg());
        fin.seekg(0, std::ios::beg);
        fin.read(&data[0], data.size());

        return data;
    }

    else
    {
        std::cout << "Failed to open " << filename << ".\n";
        exit(1);
    }

}
```

这个函数如我的注释所写，仅从我们的目标文件中检索一个字节的串，在我们的测试情况下，这个文件仍将为`Canon_40D.jpg`

```c++
//
// this will take 1% of the bytes from our valid jpeg and
// flip a random bit in the byte and return the altered string
//
std::string bit_flip(std::string data)
{
    
    int size = (data.length() - 4);
    int num_of_flips = (int)(size * .01);

    // get a vector full of 1% of random byte indexes
    std::vector<int> picked_indexes;
    for (int i = 0; i < num_of_flips; i++)
    {
        int picked_index = rand() % size;
        picked_indexes.push_back(picked_index);
    }

    // iterate through the data string at those indexes and flip a bit
    for (int i = 0; i < picked_indexes.size(); ++i)
    {
        int index = picked_indexes[i];
        char current = data.at(index);
        int decimal = ((int)current & 0xff);
        
        int bit_to_flip = rand() % 8;
        
        decimal ^= 1 << bit_to_flip;
        decimal &= 0xff;
        
        data[index] = (char)decimal;
    }

    return data;

}
```

这个函数直接等同于我们`Python`脚本中的`bit_flip()`函数

```c++
//
// takes mutated string and creates new jpeg with it;
//
void create_new(std::string mutated)
{
    std::ofstream fout("mutated.jpg", std::ios::binary);

    if (fout.is_open())
    {
        fout.seekp(0, std::ios::beg);
        fout.write(&mutated[0], mutated.size());
    }
    else
    {
        std::cout << "Failed to create mutated.jpg" << ".\n";
        exit(1);
    }

}
```

这个函数将简单地创建一个`mutated.jpg`文件，跟我们`Python`脚本中的`create_new()`函数类似

```c++
//
// function to run a system command and store the output as a string;
// https://www.jeremymorgan.com/tutorials/c-programming/how-to-capture-the-output-of-a-linux-command-in-c/
//
std::string get_output(std::string cmd)
{
    std::string output;
    FILE * stream;
    char buffer[256];

    stream = popen(cmd.c_str(), "r");
    if (stream)
    {
        while (!feof(stream))
            if (fgets(buffer, 256, stream) != NULL) output.append(buffer);
                pclose(stream);
    }

    return output;

}

//
// we actually run our exiv2 command via the get_output() func;
// retrieve the output in the form of a string and then we can parse the string;
// we'll save all the outputs that result in a segfault or floating point except;
//
void exif(std::string mutated, int counter)
{
    std::string command = "exif mutated.jpg -verbose 2>&1";

    std::string output = get_output(command);

    std::string segfault = "Segmentation";
    std::string floating_point = "Floating";

    std::size_t pos1 = output.find(segfault);
    std::size_t pos2 = output.find(floating_point);

    if (pos1 != -1)
    {
        std::cout << "Segfault!\n";
        std::ostringstream oss;
        oss << "/root/cppcrashes/crash." << counter << ".jpg";
        std::string filename = oss.str();
        std::ofstream fout(filename, std::ios::binary);

        if (fout.is_open())
            {
                fout.seekp(0, std::ios::beg);
                fout.write(&mutated[0], mutated.size());
            }
        else
        {
            std::cout << "Failed to create " << filename << ".jpg" << ".\n";
            exit(1);
        }
    }
    else if (pos2 != -1)
    {
        std::cout << "Floating Point!\n";
        std::ostringstream oss;
        oss << "/root/cppcrashes/crash." << counter << ".jpg";
        std::string filename = oss.str();
        std::ofstream fout(filename, std::ios::binary);

        if (fout.is_open())
            {
                fout.seekp(0, std::ios::beg);
                fout.write(&mutated[0], mutated.size());
            }
        else
        {
            std::cout << "Failed to create " << filename << ".jpg" << ".\n";
            exit(1);
        }
    }
}
```

这两个函数一起工作，`get_output`函数接收一个`C++`字符串作为参数，将在操作系统上运行该命令并捕获输出，然后，该函数将输出作为一个字符串返回给调用函数`exif()`

`exif()`将拿到输出并寻找`Segmentation fault`或`Floating point exception`错误，如果发现这些输出，就把这些字节写入一个文件并保存为`crash.<counter>.jpg`文件，与我们的`Python fuzzer`非常相似。

```c++
//
// simply generates a vector of strings that are our 'magic' values;
//
std::vector<std::string> vector_gen()
{
    std::vector<std::string> magic;

    using namespace std::string_literals;

    magic.push_back("\xff");
    magic.push_back("\x7f");
    magic.push_back("\x00"s);
    magic.push_back("\xff\xff");
    magic.push_back("\x7f\xff");
    magic.push_back("\x00\x00"s);
    magic.push_back("\xff\xff\xff\xff");
    magic.push_back("\x80\x00\x00\x00"s);
    magic.push_back("\x40\x00\x00\x00"s);
    magic.push_back("\x7f\xff\xff\xff");

    return magic;
}

//
// randomly picks a magic value from the vector and overwrites that many bytes in the image;
//
std::string magic(std::string data, std::vector<std::string> magic)
{
    
    int vector_size = magic.size();
    int picked_magic_index = rand() % vector_size;
    std::string picked_magic = magic[picked_magic_index];
    int size = (data.length() - 4);
    int picked_data_index = rand() % size;
    data.replace(picked_data_index, magic[picked_magic_index].length(), magic[picked_magic_index]);

    return data;

}

//
// returns 0 or 1;
//
int func_pick()
{
    int result = rand() % 2;

    return result;
}
```

这些函数与我们的`Python`实现也很相似，`vector_gen()`几乎只是创建了我们的`magic value`的`vector`，然后像`magic()`这样的后续函数使用该`vector`随机选择一个`index`，并用相应的变异数据覆盖有效`jpeg`中的数据。

`func_pick()`非常简单，只是返回一个0或一个1，这样我们的`fuzzer`就可以随机地选择`bit_flip()`或`magic()`来变异我们的有效的`jpeg`文件，为了保持一致性，让我们的`fuzzer`暂时只选择`bit_flip()`，在我们的程序中添加一行临时的`function = 1`，这样我们就能与我们的`Python`测试相匹配了

下面就是我们的`main()`函数，它执行了我们到目前为止的所有代码：

```c++
int main(int argc, char** argv)
{

    if (argc < 3)
    {
        std::cout << "Usage: ./cppfuzz <valid jpeg> <number_of_fuzzing_iterations>\n";
        std::cout << "Usage: ./cppfuzz Canon_40D.jpg 10000\n";
        return 1;
    }

    // start timer
    auto start = std::chrono::high_resolution_clock::now();

    // initialize our random seed
    srand((unsigned)time(NULL));

    // generate our vector of magic numbers
    std::vector<std::string> magic_vector = vector_gen();

    std::string filename = argv[1];
    int iterations = atoi(argv[2]);

    int counter = 0;
    while (counter < iterations)
    {

        std::string data = get_bytes(filename);

        int function = func_pick();
        function = 1;
        if (function == 0)
        {
            // utilize the magic mutation method; create new jpg; send to exiv2
            std::string mutated = magic(data, magic_vector);
            create_new(mutated);
            exif(mutated,counter);
            counter++;
        }
        else
        {
            // utilize the bit flip mutation; create new jpg; send to exiv2
            std::string mutated = bit_flip(data);
            create_new(mutated);
            exif(mutated,counter);
            counter++;
        }
    }

    // stop timer and print execution time
    auto stop = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(stop - start);
    std::cout << "Execution Time: " << duration.count() << "ms\n";

    return 0;
}
```

我们从命令行参数中得到一个有效的`JPEG`进行变异和`fuzzing`迭代的次数，我们用`std::chrono`命名空间建立了一些计时机制，以计时我们的程序需要运行多长时间。

我们在这里只选择`bit_flip()`类型的变异是一种作弊，但是这要是我们在`Python`所做地一样，所以我们想要的一种公平地比较。

让我们继续然后运行它进行`100k`次迭代，并且与`Python fuzzer`基准测试`256s`进行比较。

当我们运行`C++ fuzzer`时，我们得到了一个用毫秒为单位的打印时间：`Execution Time: 172638ms` 相当于 `172s`

因此，我们用我们新的`C++ fuzzer`轻松地摧毁了我们的`Python fuzzer`！让我们继续在这里做一些算数运算`172/256 = 67%`，因此我们用`C++`实现的速度大约是快了`33%`


让我们带着我们优化过的`Python`和`C++ fuzzer`，去尝试一个新的目标吧!


## Selecting a New Victim

看一下`Kali Linux`上预装的东西，因为这是我们的操作环境，让我们看一下`exiv2`，它是在`/usr/bin/exiv2`中被找到。

```
root@kali:~# exiv2 -h
Usage: exiv2 [ options ] [ action ] file ...

Manipulate the Exif metadata of images.

Actions:
  ad | adjust   Adjust Exif timestamps by the given time. This action
                requires at least one of the -a, -Y, -O or -D options.
  pr | print    Print image metadata.
  rm | delete   Delete image metadata from the files.
  in | insert   Insert metadata from corresponding *.exv files.
                Use option -S to change the suffix of the input files.
  ex | extract  Extract metadata to *.exv, *.xmp and thumbnail image files.
  mv | rename   Rename files and/or set file timestamps according to the
                Exif create timestamp. The filename format can be set with
                -r format, timestamp options are controlled with -t and -T.
  mo | modify   Apply commands to modify (add, set, delete) the Exif and
                IPTC metadata of image files or set the JPEG comment.
                Requires option -c, -m or -M.
  fi | fixiso   Copy ISO setting from the Nikon Makernote to the regular
                Exif tag.
  fc | fixcom   Convert the UNICODE Exif user comment to UCS-2. Its current
                character encoding can be specified with the -n option.

Options:
   -h      Display this help and exit.
   -V      Show the program version and exit.
   -v      Be verbose during the program run.
   -q      Silence warnings and error messages during the program run (quiet).
   -Q lvl  Set log-level to d(ebug), i(nfo), w(arning), e(rror) or m(ute).
   -b      Show large binary values.
   -u      Show unknown tags.
   -g key  Only output info for this key (grep).
   -K key  Only output info for this key (exact match).
   -n enc  Charset to use to decode UNICODE Exif user comments.
   -k      Preserve file timestamps (keep).
   -t      Also set the file timestamp in 'rename' action (overrides -k).
   -T      Only set the file timestamp in 'rename' action, do not rename
           the file (overrides -k).
   -f      Do not prompt before overwriting existing files (force).
   -F      Do not prompt before renaming files (Force).
   -a time Time adjustment in the format [-]HH[:MM[:SS]]. This option
           is only used with the 'adjust' action.
   -Y yrs  Year adjustment with the 'adjust' action.
   -O mon  Month adjustment with the 'adjust' action.
   -D day  Day adjustment with the 'adjust' action.
   -p mode Print mode for the 'print' action. Possible modes are:
             s : print a summary of the Exif metadata (the default)
             a : print Exif, IPTC and XMP metadata (shortcut for -Pkyct)
             t : interpreted (translated) Exif data (-PEkyct)
             v : plain Exif data values (-PExgnycv)
             h : hexdump of the Exif data (-PExgnycsh)
             i : IPTC data values (-PIkyct)
             x : XMP properties (-PXkyct)
             c : JPEG comment
             p : list available previews
             S : print structure of image
             X : extract XMP from image
   -P flgs Print flags for fine control of tag lists ('print' action):
             E : include Exif tags in the list
             I : IPTC datasets
             X : XMP properties
             x : print a column with the tag number
             g : group name
             k : key
             l : tag label
             n : tag name
             y : type
             c : number of components (count)
             s : size in bytes
             v : plain data value
             t : interpreted (translated) data
             h : hexdump of the data
   -d tgt  Delete target(s) for the 'delete' action. Possible targets are:
             a : all supported metadata (the default)
             e : Exif section
             t : Exif thumbnail only
             i : IPTC data
             x : XMP packet
             c : JPEG comment
   -i tgt  Insert target(s) for the 'insert' action. Possible targets are
           the same as those for the -d option, plus a modifier:
             X : Insert metadata from an XMP sidecar file <file>.xmp
           Only JPEG thumbnails can be inserted, they need to be named
           <file>-thumb.jpg
   -e tgt  Extract target(s) for the 'extract' action. Possible targets
           are the same as those for the -d option, plus a target to extract
           preview images and a modifier to generate an XMP sidecar file:
             p[<n>[,<m> ...]] : Extract preview images.
             X : Extract metadata to an XMP sidecar file <file>.xmp
   -r fmt  Filename format for the 'rename' action. The format string
           follows strftime(3). The following keywords are supported:
             :basename:   - original filename without extension
             :dirname:    - name of the directory holding the original file
             :parentname: - name of parent directory
           Default filename format is %Y%m%d_%H%M%S.
   -c txt  JPEG comment string to set in the image.
   -m file Command file for the modify action. The format for commands is
           set|add|del <key> [[<type>] <value>].
   -M cmd  Command line for the modify action. The format for the
           commands is the same as that of the lines of a command file.
   -l dir  Location (directory) for files to be inserted from or extracted to.
   -S .suf Use suffix .suf for source files for insert command.
```

看一下帮助指南，让我们继续随机地尝试一下用于`Print image metadata`的`pr`和用于`Be verbose during the program run`的`-v`指令，你可以从这个帮助指南中看到这里有大量的攻击面供我们探索，但现在让我们把事情简单化。

现在在我们`fuzzer`中的命令行指令字符将是这样的：`exiv2 pr -v mutated.jpg`

我们继续更新我们的`fuzzer`，看看我们是否能在一个更难的目标上找到`bug`，值得一提的是，这个目标是受支持的，而不像我们上一个目标，去找一个微不足道的二进制的`bugs`（`Github`上不再被支持的7年已久的项目）

这个目标已经被更高级的`fuzzer`进行模糊测试过了，你可以简单地通过谷歌`ASan exiv2`搜素之类地东西，并获得大量的`fuzzer`在二进制文件上创建的`segfaults`，并转发`ASan`输出到`github`仓库作为一个`bug`，这是我们上一个目标的重要一步。

[exiv2 on Github](https://github.com/Exiv2/exiv2)

[exiv2 Website](https://www.exiv2.org/)

## Fuzzing Our New Target

让我们从我们新的和改进的`Python fuzzer`开始，监测它在`50k`次迭代中的性能，让我们添加一些代码，除了我们的`Segmentation fault detection`外，还监测`Floating point exceptions`（称之为赌怪！），我们的新`exif()`函数将看起来像这样：

```python
def exif(counter,data):

    p = Popen(["exiv2", "pr", "-v", "mutated.jpg"], stdout=PIPE, stderr=PIPE)
    (out,err) = p.communicate()

    if p.returncode == -11:
        f = open("crashes2/crash.{}.jpg".format(str(counter)), "ab+")
        f.write(data)
        print("Segfault!")

    elif p.returncode == -8:
        f = open("crashes2/crash.{}.jpg".format(str(counter)), "ab+")
        f.write(data)
        print("Floating Point!")
```

我们来看看`python3 -m cProfile -s cumtime subpro.py ~/jpegs/Canon_40D.jpg`的输出：

```
75780446 function calls (75780309 primitive calls) in 213.595 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
     15/1    0.000    0.000  213.595  213.595 {built-in method builtins.exec}
        1    1.481    1.481  213.595  213.595 subpro.py:3(<module>)
    50000    0.818    0.000  187.205    0.004 subpro.py:111(exif)
    50000    0.543    0.000  143.499    0.003 subprocess.py:920(communicate)
    50000    6.773    0.000  142.873    0.003 subprocess.py:1662(_communicate)
  1641352    3.186    0.000  122.668    0.000 selectors.py:402(select)
  1641352  118.799    0.000  118.799    0.000 {method 'poll' of 'select.poll' objects}
    50000    1.220    0.000   42.888    0.001 subprocess.py:681(__init__)
    50000    4.400    0.000   39.364    0.001 subprocess.py:1412(_execute_child)
  1691919   25.759    0.000   25.759    0.000 {built-in method posix.read}
    50000    3.863    0.000   13.938    0.000 subpro.py:14(bit_flip)
  7950000    3.587    0.000    9.991    0.000 random.py:256(choice)
    50000    7.495    0.000    7.495    0.000 {built-in method _posixsubprocess.fork_exec}
    50000    0.148    0.000    7.081    0.000 subpro.py:105(create_new)
  7950000    3.884    0.000    5.764    0.000 random.py:224(_randbelow)
   200000    4.582    0.000    4.582    0.000 {built-in method io.open}
    50000    4.192    0.000    4.192    0.000 {method 'close' of '_io.BufferedRandom' objects}
    50000    1.339    0.000    3.612    0.000 os.py:617(get_exec_path)
    50000    1.641    0.000    3.309    0.000 subpro.py:8(get_bytes)
   100000    0.077    0.000    1.822    0.000 subprocess.py:1014(wait)
   100000    0.432    0.000    1.746    0.000 subprocess.py:1621(_wait)
   100000    0.256    0.000    1.735    0.000 selectors.py:351(register)
   100000    0.619    0.000    1.422    0.000 selectors.py:234(register)
   350000    0.380    0.000    1.402    0.000 subprocess.py:1471(<genexpr>)
 12066004    1.335    0.000    1.335    0.000 {method 'getrandbits' of '_random.Random' objects}
    50000    0.063    0.000    1.222    0.000 subprocess.py:1608(_try_wait)
    50000    1.160    0.000    1.160    0.000 {built-in method posix.waitpid}
   100000    0.519    0.000    1.143    0.000 os.py:674(__getitem__)
  1691352    0.902    0.000    1.097    0.000 selectors.py:66(__len__)
  7234121    1.023    0.000    1.023    0.000 {method 'append' of 'list' objects}
-----SNIP-----
```

看起来我们总共花了`213s`，但并没有真正发现任何`bug`，这很遗憾，但可能只是运气原因，让我们在同样的情况下运行我们的`C++ fuzzer`，并监测其输出。

在这里，我们得到了一个差不多的时间，但有很大的提升。

```
root@kali:~# ./blogcpp ~/jpegs/Canon_40D.jpg 50000
Execution Time: 170829ms
```

这是一个相当大的提升，43秒，这缩短了我们的`Python fuzzer`20%的时间

让我们继续跑一段时间的`C++ fuzzer`，看看是否我们能找到任何`bugs` :)


## Bugs on Our New Target!

在再次运行`fuzzer`大约`10s`后，我得到了以下这个终端输出：

```
root@kali:~# ./blogcpp ~/jpegs/Canon_40D.jpg 1000000
Floating Point!
```

这似乎我们满足了`a Floating Point exception`的需求

我们应该有一个很棒的`jpg`在`cppcrashes`文件夹中等着我们

```
root@kali:~/cppcrashes# ls
crash.522.jpg
```

让我们通过对这个样本运行`exiv2`来证实这个错误。

```
root@kali:~/cppcrashes# exiv2 pr -v crash.522.jpg
File 1/1: crash.522.jpg
Error: Offset of directory Image, entry 0x011b is out of bounds: Offset = 0x080000ae; truncating the entry
Warning: Directory Image, entry 0x8825 has unknown Exif (TIFF) type 68; setting type size 1.
Warning: Directory Image, entry 0x8825 doesn't look like a sub-IFD.
File name       : crash.522.jpg
File size       : 7958 Bytes
MIME type       : image/jpeg
Image size      : 100 x 68
Camera make     : Aanon
Camera model    : Canon EOS 40D
Image timestamp : 2008:05:30 15:56:01
Image number    : 
Exposure time   : 1/160 s
Aperture        : F7.1
Floating point exception
```

我们确实发现了一个新的`bug`! 这实在是太令人兴奋了，我们应该向`Github`上的`exiv2`开发者发布一份`bug report`

为了找点有趣的，让我们比较一下我们的原始`fuzzer`在50,000次迭代中的表现：

```
123052109 function calls (123001828 primitive calls) in 6243.939 seconds
```

正如你所看到的，`6243s`明显比我们的`C++ fuzzer`基准的`170s`慢

## Addendum 15/May/2020

仅试试通过移植`C++ fuzzer`到`C`上，我自己也做了一些适当的改进，其中一个我所修改的逻辑为仅从原始有效的图像中收集一次数据，然后在每次`fuzzing`迭代把数据复制到新分配的缓冲区，之后再在新分配的缓冲区中执行变异操作，这个与`C++ fuzzer`基本相同的`C`版本比`C++`执行的效果更好，这是这两个迭代`200k`次之间的对比（你可以忽略这个`crash`的发现数，因为这个`fuzzer`是相当愚蠢且100%随机的）：

```
h0mbre:~$ time ./cppfuzz Canon_40D.jpg 200000
<snipped_results>

real    10m45.371s
user    7m14.561s
sys     3m10.529s

h0mbre:~$ time ./cfuzz Canon_40D.jpg 200000
<snipped_results>

real    10m7.686s
user    7m27.503s
sys     2m20.843s
```

因此，在超过`200k`次的迭代中，我们以节省`35-40s`结束改进，这个在我们的测试中是十分典型的，因此，仅仅通过少量的逻辑更改和使用较少的`C++`提供的抽象接口中，我们节省了大量的`sys`的时间，我们的速度大概提升了`5%`

### Monitoring Child Process Exit Status

在完成`C`的翻译后，我去推特询问了有关性能改进的建议，[@lcamtuf](https://twitter.com/lcamtuf)，`AFL`的创建者，向我解释说，我不应该在我的代码中用`popen`，因为它会生成一个`shell`并且性能很差，这是我寻求帮助的代码片段：

```c
void exif(int iteration) {
    
    FILE *fileptr;
    
    //fileptr = popen("exif_bin target.jpeg -verbose >/dev/null 2>&1", "r");
    fileptr = popen("exiv2 pr -v mutated.jpeg >/dev/null 2>&1", "r");

    int status = WEXITSTATUS(pclose(fileptr));
    switch(status) {
        case 253:
            break;
        case 0:
            break;
        case 1:
            break;
        default:
            crashes++;
            printf("\r[>] Crashes: %d", crashes);
            fflush(stdout);
            char command[50];
            sprintf(command, "cp mutated.jpeg ccrashes/crash.%d.%d",
             iteration,status);
            system(command);
            break;
    }
}
```

正如你所看到的，我们使用`popen()`，运行一个`shell-command`，然后关闭子进程的文件指针，返回用`WEXITSTATUS`宏监控的`exit-status`，我正在过滤掉一些我不关心的`exit codes`，如253、0和1，并希望看到一些与我们已经用`C++ fuzzer`发现的`floating point errors`有关的代码，或甚至可能是一个`segfault`，`@lcamtuf`建议我不使用`popen()`，而是调用`fork()`来生成一个子进程，`execvp()`让子进程执行一个命令，最后使用`waitpid()`来等待子进程终止并返回退出状态。

由于我们在这个`syscall`路径中没有一个合适的`shell`，我不得不同时打开一个`/dev/null`的句柄，并调用`dup2()`路由`stdout`和`stderr`到`/dev/null`，因为我们并不关心命令输出，我还使用了`WTERMSIG`宏来检索在`WIFSIGNALED`宏返回真时终止子进程的信号，这将表明我们得到了一个`segfault`或`floating point exception`，等等，所以现在，我们更新的函数看起来像这样：

```c
void exif(int iteration) {
    
    char* file = "exiv2";
    char* argv[4];
    argv[0] = "pr";
    argv[1] = "-v";
    argv[2] = "mutated.jpeg";
    argv[3] = NULL;
    pid_t child_pid;
    int child_status;

    child_pid = fork();
    if (child_pid == 0) {
        // this means we're the child process
        int fd = open("/dev/null", O_WRONLY);

        // dup both stdout and stderr and send them to /dev/null
        dup2(fd, 1);
        dup2(fd, 2);
        close(fd);

        execvp(file, argv);
        // shouldn't return, if it does, we have an error with the command
        printf("[!] Unknown command for execvp, exiting...\n");
        exit(1);
    }
    else {
        // this is run by the parent process
        do {
            pid_t tpid = waitpid(child_pid, &child_status, WUNTRACED |
             WCONTINUED);
            if (tpid == -1) {
                printf("[!] Waitpid failed!\n");
                perror("waitpid");
            }
            if (WIFEXITED(child_status)) {
                //printf("WIFEXITED: Exit Status: %d\n", WEXITSTATUS(child_status));
            } else if (WIFSIGNALED(child_status)) {
                crashes++;
                int exit_status = WTERMSIG(child_status);
                printf("\r[>] Crashes: %d", crashes);
                fflush(stdout);
                char command[50];
                sprintf(command, "cp mutated.jpeg ccrashes/%d.%d", iteration, 
                exit_status);
                system(command);
            } else if (WIFSTOPPED(child_status)) {
                printf("WIFSTOPPED: Exit Status: %d\n", WSTOPSIG(child_status));
            } else if (WIFCONTINUED(child_status)) {
                printf("WIFCONTINUED: Exit Status: Continued.\n");
            }
        } while (!WIFEXITED(child_status) && !WIFSIGNALED(child_status));
    }
}
```

你可以看到，这极大地提高了我们`200k`次迭代基准测试的性能：

```
h0mbre:~$ time ./cfuzz2 Canon_40D.jpg 200000
<snipped_results>

real    8m30.371s
user    6m10.219s
sys     2m2.098s
```

### Summary of Results

+ C++ Fuzzer – 310 iterations/sec
+ C Fuzzer – 329 iterations/sec (+ 6%)
+ C Fuzzer 2.0 – 392 iterations/sec (+ 26%)

感谢[@lcamtuf](https://twitter.com/lcamtuf)和[@carste1n](https://twitter.com/carste1n)的帮助

我已经把代码上传到`https://github.com/h0mbre/Fuzzing/tree/master/JPEGMutation`