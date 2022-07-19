---
title: '对wps的qtcore的某一接口fuzz'
date: 2022-03-29 09:28:26
category: Fuzz
tags: [afl++, qemu_mode]
published: true
hideInList: false
feature: 
isTop: false
---

主要看到以下三篇`blog`
> http://zeifan.my/security/rce/heap/2020/09/03/wps-rce-heap.html
> https://www.anquanke.com/post/id/240938
> https://ruan777.github.io/2021/06/02/使用winafl对qtcore的一次fuzz尝试

因此尝试对`linux`上的`wps`的`qtcore4`进行`fuzz`

```
环境：
linux: ubuntu 20.04
wps: 11.1.0.10161
libcQtCore: 4.7.4
```

## 整体的逻辑

根据`Nafiez`的报告，可以知道，主要是`kso.dll`中调用`QtCore4.dll`的`QImageReader::read()`出错的，因此后续两篇文章均对于`QtCore4.dll`的该接口进行`fuzz`

```
0:000> !heap -p -a cc53afbc
    address cc53afbc found in
    _DPH_HEAP_ROOT @ 6731000
    in busy allocation (  DPH_HEAP_BLOCK:         UserAddr         UserSize -         VirtAddr         VirtSize)
                                cc36323c:         cc53afa8               58 -         cc53a000             2000
    6f13ab70 verifier!AVrfDebugPageHeapAllocate+0x00000240
    77a9909b ntdll!RtlDebugAllocateHeap+0x00000039
    779ebbad ntdll!RtlpAllocateHeap+0x000000ed
    779eb0cf ntdll!RtlpAllocateHeapInternal+0x0000022f
    779eae8e ntdll!RtlAllocateHeap+0x0000003e
    6f080269 MSVCR100!malloc+0x0000004b
    6f08233b MSVCR100!operator new+0x0000001f
    6b726c67 QtCore4!QImageData::create+0x000000fa
    6b726b54 QtCore4!QImage::QImage+0x0000004e
    6b7a0e21 QtCore4!png_get_text+0x00000436
    6b79d7a8 QtCore4!QImageIOHandler::setFormat+0x000000de
    6b79d457 QtCore4!QPixmapData::fromFile+0x000002bf
    6b725eb4 QtCore4!QImageReader::read+0x000001e2
    6d0ca585 kso!kpt::VariantImage::forceUpdateCacheImage+0x0000254e
    6d0c5964 kso!kpt::Direct2DPaintEngineHelper::operator=+0x00000693
    6d0c70d0 kso!kpt::RelativeRect::unclipped+0x00001146
    6d0c8d0c kso!kpt::VariantImage::forceUpdateCacheImage+0x00000cd5
    6d451d5c kso!BlipCacheMgr::BrushCache+0x0000049a
    6d451e85 kso!BlipCacheMgr::GenerateBitmap+0x0000001d
    6d453227 kso!BlipCacheMgr::GenCachedBitmap+0x00000083
    6d29bb92 kso!drawing::PictureRenderLayer::render+0x000009b6
    6d450fb1 kso!drawing::RenderTargetImpl::paint+0x00000090
    6d29b528 kso!drawing::PictureRenderLayer::render+0x0000034c
    6d2a2d83 kso!drawing::VisualRenderer::render+0x00000060
    6d2b8970 kso!drawing::SingleVisualRenderer::drawNormal+0x000002b5
    6d2b86a7 kso!drawing::SingleVisualRenderer::draw+0x000001e1
    6d2b945e kso!drawing::SingleVisualRenderer::draw+0x00000046
    6d3d0142 kso!drawing::ShapeVisual::paintEvent+0x0000044a
    680a2b5c wpsmain!WpsShapeTreeVisual::getHittestSubVisuals+0x000068f1
    6d0e36df kso!AbstractVisual::visualEvent+0x00000051
    6d3cbe97 kso!drawing::ShapeVisual::visualEvent+0x0000018f
    6d0eba90 kso!VisualPaintEvent::arriveVisual+0x0000004e
```

后续两篇文章`fuzz`的代码逻辑为

```
QImage image;
QImageReader reader;
QString image_file_name;

# transform (char[] file_name) to (QString image_file_name)
reader.setFileName(image_file_name)
reader.read(image);
```

## 从windows到linux

在迁移的时候，就去找了一下对应的`libQtCore.so.4.7.4`的接口，得到了以下的代码

```c
// gcc -g -masm=intel ./qt_reader.c -ldl -no-pie -o qt_reader

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <dlfcn.h>

int main(int argc, char* argv[])
{
    dlopen("/path/libc++.so.1", RTLD_LAZY);
    dlopen("/path/libpng12.so.0", RTLD_LAZY);
    dlopen("/path/libc++abi.so.1", RTLD_LAZY);
    void* handle = dlopen("/path/libQtCore.so.4.7.4", RTLD_LAZY);

    if (handle == 0)
    {
        puts("open handle failed");
        printf("%s\n", dlerror());
        exit(-1);
    }
    
    // QImageReader::QImageReader(QImageReader *this)
    void (*qt_qimageReader)() = dlsym(handle, "_ZN12QImageReaderC2Ev");

    // QImageReader::read(QImageReader *this, QImage *a2)
    void (*qt_qimageReader_read)() = dlsym(handle, "_ZN12QImageReader4readEP6QImage");

    // QImageReader::setFileName(QImageReader *this, const QString *a2)
    void (*qt_setFileName)() = dlsym(handle, "_ZN12QImageReader11setFileNameERK7QString");

    // QString::QString(QString *this, const QChar *)
    void (*qt_qstring)() = dlsym(handle, "_ZN7QStringC2EPK5QChar");

    // QString::fromLatin1(QString *this, const char *, unsigned int)
    void (*qt_qstring_fromlatin1)() = dlsym(handle, "_ZN7QString10fromLatin1EPKci");

    // QImage::QImage(QImage *this)
    void (*qt_qimage)() = dlsym(handle, "_ZN6QImageC2Ev");

    // QFile::exists(QFile *__hidden this)
    void (*qt_qfile_exits)() = dlsym(handle, "_ZNK5QFile6existsEv");

    // QFile::close()
    void (*qt_close)() = dlsym(handle, "_ZN5QFile5closeEv");

    char* image = (char*)malloc(0x400);
	char* qstring = (char*)malloc(0x400);
	char* reader = (char*)malloc(0x400);

    char* file_name = argv[1];

    // pusha
    __asm__ __volatile__(
        "push rax\n"
        "push rbx\n"
        "push rcx\n"  
        "push rdx\n"      
        "push rbp\n"      
        "push rdi\n"      
        "push rsi\n"      
        "push r8\n"      
        "push r9\n"      
        "push r10\n"      
        "push r11\n"      
        "push r12\n"      
        "push r13\n"      
        "push r14\n"      
        "push r15\n"
        "push r15\n");

    // QImage::QImage(image)
    __asm__ __volatile__ (
        "call rax\n"
        :
        : "D"(image), "a"(qt_qimage)
    );
    
    // QImageReader::QImageReader(reader)
    __asm__ __volatile__ (
        "call rax\n"
        :
        : "D"(reader), "a"(qt_qimageReader)
    );

    // **************** This is a wrong interface of transforming the char to QString ***********************
    // // QString::QString(qstring, file_name)
    // __asm__ __volatile__ (
    //     "call rax\n"
    //     :
    //     : "D"(qstring), "S"(file_name), "a"(qt_qstring)
    // );

    // QString::fromLatin1(QString *this, const char *, unsigned int)
    __asm__ __volatile__ (
        "call rax\n"
        :
        : "D"(qstring), "S"(file_name), "d"(strlen(file_name)), "a"(qt_qstring_fromlatin1)
    );

    // QImageReader::setFileName(reader, qstring)
    __asm__ __volatile__ (
        "call rax\n"
        :
        : "D"(reader), "S"(qstring), "a"(qt_setFileName)
    );

    // QFile::exits(qfile) qfile = reader+0x10
    __asm__ __volatile__ (
        "mov rsi, [rdi]\n"
        "mov rdi, [rsi+0x10]\n"
        "call rax\n"
        "test al, al\n"
        "je error\n"
        :
        : "D"(reader), "a"(qt_qfile_exits)
    );

    puts("file exists");

    // QImageReader::read(reader, qimage)
    __asm__ __volatile__ (
        "call rax\n"
        :
        : "D"(reader), "S"(image), "a"(qt_qimageReader_read)
    );
    
    // QFile::close()
    __asm__ __volatile__ (
        "mov rsi, [rdi]\n"
        "mov rdi, [rsi+0x10]\n"
        "call rax\n"
        "jmp out\n"
        :
        : "D"(reader), "a"(qt_close)
    );

    // error:
    __asm__ __volatile__ (
        "error:\n"
    );
    puts("error: file not exists");

    // popa
    __asm__ __volatile__(
        "out:\n"
        "pop r15\n"
        "pop r15\n"
        "pop r14\n"
        "pop r13\n"
        "pop r12\n"
        "pop r11\n"
        "pop r10\n"
        "pop r9\n"
        "pop r8\n"
        "pop rsi\n"
        "pop rdi\n"
        "pop rbp\n"
        "pop rdx\n"
        "pop rcx\n"
        "pop rbx\n"
        "pop rax\n"
        );

    puts("Done");

    return 0;
}


```

这个地方，最开始写的时候，撸了一版跟后两篇一样的接口的代码，但是发现覆盖率并没有上升，最后调的时候，发现在`reader.read()`的第一次判断支持的文件格式（`bmp, png, jpg ...`）以及文件是否存在时，就发现程序自身就走到了`File Not found`的地方

无奈只能重新调试程序，最后找到了一个`QFile::exists()`接口，调试的时候，发现`QFile`中存储的路径的`QString`跟我用`QString::QString()`的数据结构并不一致，就换了一个`QString::fromLatin1()`接口，就能成功地跑起来了

## fuzz

在写好板子之后就想要用`afl`的`qemu_mode`进行插桩`fuzz`，折腾了半天，感觉`afl`原本的`qemu`版本以及`patch`和`libc`的接口都太老了

最后听学长的直接整`afl++`的`qemu_mode`

最初`fuzz`起来的时候，并没过多的设置，但是这样的话是全插桩，像`dlopen`的一些库都是不关心以及没必要的，而且在`afl++`的窗口也看的出来，基本上覆盖率都是不上升的，而且极低（0.10%）

利用`./afl-qemu-trace -D 1.txt -d exec,nochain ./qt_reader /tmp/1.png`记录下来`trace`，发现不同文件的`trace`差距还是很明显的，说明代码并没有写崩

最后利用`export AFL_INST_LIBS=1`给库函数也插桩之后就可以跑起来了

另外也可以通过`AFL_QEMU_INST_RANGES`设置`range`，进行范围的插桩，通过以下代码获取相关`.so`的内存地址范围

```c
struct link_map *lm = (struct link_map*)handle;
printf("base:%p\n", lm->l_addr);
```

另外翻了一下`afl`的源码，可以看到如果`cur_loc >= afl_inst_rms`则`return`，所以如果给库插桩还是要注意是否超过了`MAP_SIZE`，否则最后的比较关心的`.so`没插桩上

```c
/* Instrumentation ratio: */

static unsigned int afl_inst_rms = MAP_SIZE;

/* The equivalent of the tuple logging routine from afl-as.h. */

static inline void afl_maybe_log(abi_ulong cur_loc) {

  static __thread abi_ulong prev_loc;

  /* Optimize for cur_loc > afl_end_code, which is the most likely case on
     Linux systems. */

  if (cur_loc > afl_end_code || cur_loc < afl_start_code || !afl_area_ptr)
    return;

  /* Looks like QEMU always maps to fixed locations, so ASAN is not a
     concern. Phew. But instruction addresses may be aligned. Let's mangle
     the value to get something quasi-uniform. */

  cur_loc  = (cur_loc >> 4) ^ (cur_loc << 8);
  cur_loc &= MAP_SIZE - 1;

  /* Implement probabilistic instrumentation by looking at scrambled block
     address. This keeps the instrumented locations stable across runs. */

  if (cur_loc >= afl_inst_rms) return;

  afl_area_ptr[cur_loc ^ prev_loc]++;
  prev_loc = cur_loc >> 1;

}
```

## 其他 ~~踩坑~~~

### reader.setFileName

一开始我想去找`houjingyi`师傅是如何得到`reader.setFileName`接口在`reader.read()`之前被调用了的，因此我去尝试调试`linux`下的`wpsoffice`打开`docx`的操作

在`wpsoffice`最初并未加载`libQtCore.so.4.7.4`时，给`pthread_create`下断点，`continue`之后再给`libQtCore.so.4.7.4`中的`QImageReader.read()`和`QImageReader.setFileName()`下断点，但是其实最后并没有很明显的看出来，在调用`reader.read()`之前调用了`reader.setFileName()`

最后去询问`houjingyi`师傅，才知道，师傅是直接根据程序代码逻辑，认为`reader.read()`之前肯定有对于设置图片路径`reader.setFileName()`的操作（感觉自己的思维有点局限了，老是想明明白白调出来调用的接口和顺序，实际上全然没管开发者在开发时候的代码逻辑）

P.S. 说起来，调试 `wpsoffice` 的程序的时候，觉得特别神奇，`wpsoffice`会运行**两次**`_start`，在第一次`_start`的时候，可以看到`wpsoffice`在加载自己的程序的窗口，在第一次`_start`的最后会`jmp`第二次的`_start`，第二次的`wps`窗口，显示正在加载`docx`文件，因此当时我调试的时候，猜测第一次`_start`的时候，是在利用`QtCore`加载自己窗口，而第二次才是渲染解析`docx`文件，但是最后还是没调出来

### QString::QString

这个接口转换出来的并不是`QString`坑了我一段时间，最后找到`QFile::exists()`接口的时候，才发现，`QString::QString`转换出来的不是`QString`

### AFL_INST_LIBS AFL_QEMU_INST_RANGES

不设置这两个的话，是不会给`dlopen`打开的库函数插桩的

### fuzz 断掉继续跑

设置输入参数为`-i-` ，就可以读入 `fuzz_out/default/_resume`中的内容，继续跑

### 其他问题

顺手试了一波`Qt5.12.9`，最后跑的时候，是出现了`WARNING: Instrumentation output varies across runs.`，发现是出现了`ASLR`的情况，照理来说`qemu`应该不会出现这种情况，但是怎么解决，不太清楚

#### qasan

~~另外试了一下`qasan`也无法进行插桩检测，感觉还得再整整~~

`qasan`重新试了一下，没啥问题，至少也能跑起来，就是没啥效果，`build.py`之后，`./qasan ./elffile arg...`就可以了

#### e9afl

试了一下`e9afl`，但是觉得对于这种写的`harness`并不怎么友好，最终效果非常不好，对于库函数的插桩，应该是插桩所有函数，或者沿着接口的`CFG`需要对主要部分插桩，但是我反汇编，看的时候，感觉基本没插桩什么