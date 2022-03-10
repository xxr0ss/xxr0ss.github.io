---
title: "手把手Qiling Framework上手教程"
date: 2021-08-19T02:03:42+08:00
ShowToc: true
---

## 简介

QilingLab来源： [Shielder - QilingLab – Release](https://www.shielder.it/blog/2021/07/qilinglab-release/)，是一个包含十几个小挑战的程序，用于快速上手Qiling框架的主要功能。Qiling框架就不多做介绍了。

两个不同架构但是内容一样，任选其一做即可。aarch64版本已有Joan Sivion做的[Qiling Labs - JoanSivion Security Blog](https://joansivion.github.io/qilinglabs/)，我这里用x86-64做一遍，当作对Joan Sivion的补充和中文翻译。

## 题目样本

[qilinglab-x86_64](https://www.shielder.it/attachments/qilinglab-x86_64)

[qilinglab-aarch64](https://www.shielder.it/attachments/qilinglab-aarch64)

## 开始挑战！

qilinglab出发点只是训练qiling框架的使用，所以没有啥逆向强度，符号也没有去，我们直接用IDA打开看即可。

### 看下题目

运行看到题目列表：

```纯文本
Challenge 1: Store 1337 at pointer 0x1337.
Challenge 2: Make the 'uname' syscall return the correct values.
Challenge 3: Make '/dev/urandom' and 'getrandom' "collide".
Challenge 4: Enter inside the "forbidden" loop.
Challenge 5: Guess every call to rand().
Challenge 6: Avoid the infinite loop.
Challenge 7: Don't waste time waiting for 'sleep'.
Challenge 8: Unpack the struct and write at the target address.
Challenge 9: Fix some string operation to make the iMpOsSiBlE come true.
Challenge 10: Fake the 'cmdline' line file to return the right content.
Challenge 11: Bypass CPUID/MIDR_EL1 checks.
```


IDA打开，可以看到`main() → start()`里就是一大堆调用`challangeX(a)`并对结果进行校验，我们的目的就是让传入的指针指向的位置能被正确赋值1。没有逆向难度所以不多做解释。

直接运行的话，并不会输出每个Challenge的结果（SOLVED/UNSOLVED），甚至还会出现`segment fault`，这些都是正常现象，请放心食用，当你完成某些挑战后，自然就能看到更多的结果了。

### 基本用法

我们需要用到rootfs里的东西，所以把qiling的GitHub仓库clone下来，里面rootfs是个submodule所以干脆整个用

```bash
git clone https://github.com/qilingframework/qiling.git --recursiv
```


克隆下来



### 基本使用模板

```python
from qiling import *

def challenge1(ql: Qiling):
    pass

if __name__  == '__main__':
    path = ['qilinglab-x86_64'] # 我们的目标
    rootfs = "./qiling/examples/rootfs/x8664_linux" # 在你clone下来的仓库里
    ql = Qiling(path, rootfs)
    challenge1(ql) # 在ql.run()之前，做好我们的hook工作
    ql.run()

```




## challenge 1: 操作内存

```c
_BYTE *__fastcall challenge1(_BYTE *a1)
{
  _BYTE *result; // rax

  result = (_BYTE *)MEMORY[0x1337];
  if ( MEMORY[0x1337] == 1337 )
  {
    result = a1;
    *a1 = 1;
  }
  return result;
}
```


需要我们让内存0x1337上存放一个值为1337，我们其实并不能保证程序加载基地址，这里我们需要用

```python
ql.mem.map(0x1000, 0x1000, info='[challenge]')
```


映射一块内存，需要注意的是，qiling底层就是用的Unicorn Engine，内存映射时，要4k对齐。

综上，还是挺容易得出第一处解法。

### 脚本

```python
def challenge1(ql: Qiling):
    ql.mem.map(0x1000, 0x1000, info='[challenge1]')
    ql.mem.write(0x1337, ql.pack16(1337)) # pack16(value) == struct.pack('H', value)
```


#### Tips

在`ql.run()`前加一句`ql.verbose = 0`方便看输出内容



## challenge 2: 修改系统调用

让我们修改uname系统调用，让它返回正确的值。IDA看下“正确的值”指的是什么

```C
unsigned __int64 __fastcall challenge2(_BYTE *a1)
{
  unsigned int v2; // [rsp+10h] [rbp-1D0h]
  int v3; // [rsp+14h] [rbp-1CCh]
  int v4; // [rsp+18h] [rbp-1C8h]
  int v5; // [rsp+1Ch] [rbp-1C4h]
  struct utsname name; // [rsp+20h] [rbp-1C0h] BYREF
  char s[10]; // [rsp+1A6h] [rbp-3Ah] BYREF
  char v8[16]; // [rsp+1B0h] [rbp-30h] BYREF
  unsigned __int64 v9; // [rsp+1C8h] [rbp-18h]

  v9 = __readfsqword(0x28u);
  if ( uname(&name) )
  {
    perror("uname");
  }
  else
  {
    strcpy(s, "QilingOS");
    strcpy(v8, "ChallengeStart");
    v2 = 0;
    v3 = 0;
    while ( v4 < strlen(s) )
    {
      if ( name.sysname[v4] == s[v4] )
        ++v2;
      ++v4;
    }
    while ( v5 < strlen(v8) )
    {
      if ( name.version[v5] == v8[v5] )
        ++v3;
      ++v5;
    }
    if ( v2 == strlen(s) && v3 == strlen(v8) && v2 > 5 )
      *a1 = 1;
  }
  return __readfsqword(0x28u) ^ v9;
}
```


uname系统调用用来获取系统的一些信息，传入一个utsname结构体buffer让它填充。可以在`<sys/utsname.h>`看到utsname的定义（经过整理）：

```c
struct utsname
{
    char sysname[65];
    char nodename[65];
    char release[65];
    char version[65];
    char machine[65];
    char domainname[65];
};
```


而Qiling提供了在系统调用返回时进行hook的功能。

### 脚本

```python
def hook_uname_on_exit(ql: Qiling, *args):
    rdi = ql.reg.rdi
    ql.mem.write(rdi, b'QilingOS\x00')
    ql.mem.write(rdi + 65 * 3, b'ChallengeStart\x00')

def challenge2(ql: Qiling):
    ql.set_syscall('uname', hook_uname_on_exit, QL_INTERCEPT.EXIT)
```


## challenge 3: 劫持文件系统&系统调用

```c
unsigned __int64 __fastcall challenge3(_BYTE *a1)
{
  int v2; // [rsp+10h] [rbp-60h]
  int i; // [rsp+14h] [rbp-5Ch]
  int fd; // [rsp+18h] [rbp-58h]
  char v5; // [rsp+1Fh] [rbp-51h] BYREF
  char buf[32]; // [rsp+20h] [rbp-50h] BYREF
  char buf2[40]; // [rsp+40h] [rbp-30h] BYREF
  unsigned __int64 v8; // [rsp+68h] [rbp-8h]

  v8 = __readfsqword(0x28u);
  fd = open("/dev/urandom", 0);
  read(fd, buf, 0x20uLL);
  read(fd, &v5, 1uLL);
  close(fd);
  getrandom((__int64)buf2, 32LL, 1LL);
  v2 = 0;
  for ( i = 0; i <= 31; ++i )
  {
    if ( buf[i] == buf2[i] && buf[i] != v5 )
      ++v2;
  }
  if ( v2 == 32 )
    *a1 = 1;
  return __readfsqword(0x28u) ^ v8;
}
```


可以看到需要我们解决两个问题：

1. 直接读取`/dev/urandom`得到的随机数，和通过`getrandom()`得到的随机数要完全一样
2. 还要有一个字节的随机数，和剩下的都不一样。

Qiling提供了`QlFsMappedObject`让我们能很方便地自定义文件系统，最少要实现`close`，剩下的可以看源码，根据需要自行去实现：`read`, `write`, `fileno`, `lseek`, `fstat`, `ioctl`, `tell`, `dup`, `readline`。

### 脚本

```python
class Fake_urandom(QlFsMappedObject):
    def read(self, expected_len):
        if expected_len == 1:
            return b'\x23'  # casual single byte here
        else:
            return b'\x00' * expected_len

    def close(self):
        return 0


def hook_getrandom(ql: Qiling, buf, buflen, flags, *args):
    ql.mem.write(buf, b'\x00' * buflen)
    ql.os.set_syscall_return(0)


def challenge3(ql: Qiling):
    ql.add_fs_mapper('/dev/urandom', Fake_urandom())
    ql.set_syscall('getrandom', hook_getrandom)
```


## challenge 4: hook地址

在我的IDA上，F5看不到完整流程。

```c
__int64 challenge4()
{
  return 0LL;
}
```


所以看汇编：

![](/hands_on_qiling_framework/image/image.png "")

可以看到有个不成立的循环，正常执行不会进入。相当于是：

```c
while (i < 0) {
    *a1 = 1;
    i++;
}
return;
```


所以在比较前将终止条件改为1，这样可以执行一次，让函数参数赋值为1。



### 脚本

`hook_address`里注册的回调在执行被hook地址处之前执行，然后才执行这个地址上的指令。所以我们hook在cmp这句。这样`while`里就是比较 `i < 1`了。

```python
def enter_forbidden_loop_hook(ql: Qiling):
    ql.reg.eax = 1

def challenge4(ql: Qiling):
    """
    000055A3E4800E40 8B 45 F8   mov     eax, [rbp+var_8]
    000055A3E4800E43 39 45 FC   cmp     [rbp+var_4], eax        <<< hook here
    000055A3E4800E46 7C ED      jl      short loc_55A3E4800E35
    """
    base = ql.mem.get_lib_base(ql.path)
    hook_addr = base + 0xE43
    ql.hook_address(enter_forbidden_loop_hook, hook_addr)
```




## challenge 5: hook外部函数

来看看我们的目标：

```c
unsigned __int64 __fastcall challenge5(_BYTE *a1)
{
  unsigned int v1; // eax
  int i; // [rsp+18h] [rbp-48h]
  int j; // [rsp+1Ch] [rbp-44h]
  int v5[14]; // [rsp+20h] [rbp-40h]
  unsigned __int64 v6; // [rsp+58h] [rbp-8h]

  v6 = __readfsqword(0x28u);
  v1 = time(0LL);
  srand(v1);
  for ( i = 0; i <= 4; ++i )
  {
    v5[i] = 0;
    v5[i + 8] = rand();
  }
  for ( j = 0; j <= 4; ++j )
  {
    if ( v5[j] != v5[j + 8] )
    {
      *a1 = 0;
      return __readfsqword(0x28u) ^ v6;
    }
  }
  *a1 = 1;
  return __readfsqword(0x28u) ^ v6;
}

```


显然，只要让rand()每次都返回0是最简单的修改方法了。所以代码比较简单：

### 脚本

```python
def hook_rand(ql: Qiling):
    ql.reg.rax = 0

def challenge5(ql: Qiling):
    ql.set_api('rand', hook_rand)

```


看不到输出challenge5: SOLVED是正常的，因为卡在了challenge6，没有输出来信息。



## challenge 6: 突破死循环

流程很清晰，只要我们修改死循环部分，参数就会乖乖被赋值1

![](/hands_on_qiling_framework/image/image_1.png "")

### 脚本

同理，要抢在比较之前修改al的值

```python
def hook_while_true(ql: Qiling):
    """
    0000564846E00F12 0F B6 45 FB    movzx   eax, [rbp+var_5]
    0000564846E00F16 84 C0          test    al, al          <<< hook here
    0000564846E00F18 75 F1          jnz     short loc_564846E00F0B
    """
    ql.reg.rax = 0

def challenge6(ql: Qiling):
    base = ql.mem.get_lib_base(ql.path)
    ql.hook_address(hook_while_true, base + 0xF16)
```


## challenge 7: 不许睡了，快起来！——修改sleep()

```c
unsigned int __fastcall challenge7(_BYTE *a1)
{
  *a1 = 1;
  return sleep(0xFFFFFFFF);
}
```


可以很轻易想到n种办法：

1. 修改`sleep`参数值，减少睡眠时间
2. 自己实现api`sleep()`，直接返回
3. 修改系统调用`nanosleep()`，直接返回，因为`sleep()`底层调用的`nanosleep()`
	可以看一下
	```纯文本
	# man 3 sleep
	NOTES
	       On Linux, sleep() is implemented via nanosleep(2).   See  the  nanosleep(2)
	       man page for a discussion of the clock used. 
	```
	

#### 脚本

```python
def modify_arg(ql: Qiling):
    ql.reg.edi = 0

def i_slept_i_faked_it(ql: Qiling):
    # 我睡了，我装的
    return

def hook_nanosleep(ql: Qiling, *args, **kwargs):
    # 注意参数列表
    return

def challenge7(ql: Qiling):
    # method 1
    # ql.set_api('sleep', modify_arg, QL_INTERCEPT.ENTER)
    # method 2
    # ql.set_api('sleep', i_slept_i_faked_it)
    # method 3
    ql.set_syscall('nanosleep', hook_nanosleep)
```


## challenge 8: 解析结构体，往正确地址写入值

这里直接借用Joan Sivion整理的代码：

```c
void challenge8(char *check) {
    random_struct *s;

    s = (random_struct *)malloc(24);
    s->some_string = (char *)malloc(0x1E);
    s->magic = 0x3DFCD6EA00000539;
    strcpy(s->field_0, "Random data");
    s->check_addr = check;
}

struct random_struct {
  char *some_string;
  __int64 magic;
  char *check_addr;
};
```


方法有两种：

1. 最后一句的时候，获取这个结构体地址，可以看一下最后部分的汇编：
	```纯文本
	.text:0000564846E00FA9 48 8B 45 F8      mov     rax, [rbp+stru] ; stru == -8
	.text:0000564846E00FAD 48 8B 55 E8      mov     rdx, [rbp+var_18]
	.text:0000564846E00FB1 48 89 50 10      mov     [rax+10h], rdx
	.text:0000564846E00FB5 90               nop
	.text:0000564846E00FB6 C9               leave
	.text:0000564846E00FB7 C3               retn
	```
	
	然后结构体偏移`+0x16`的地方就是保存了参数的位置。
2. 使用qiling提供的机制，用`ql.mem.search()`搜索内存。看一下代码可以发现，这个结构体保存了一个魔数`0x3DFCD6EA00000539`，这正好是个特征方便我们去搜索这块内存。当然，还在第一个字段保存了一个指针指向一个固定的字符串，如果那个魔数在内存中出现多次，那显然我们还需要进一步获取信息确定是不是我们要找的那个结构体：保存了参数`check`的结构体。

### 脚本

```python
def hook_struct(ql: Qiling):
    """
    0000564846E00FA9 48 8B 45 F8      mov     rax, [rbp+str]    ; rbp - 8
    0000564846E00FAD 48 8B 55 E8      mov     rdx, [rbp+var_18]
    0000564846E00FB1 48 89 50 10      mov     [rax+10h], rdx
    0000564846E00FB5 90               nop       <<< hook here
    0000564846E00FB6 C9               leave
    0000564846E00FB7 C3               retn
    """
    heap_struct_addr = ql.unpack64(ql.mem.read(ql.reg.rbp - 8, 8))
    heap_struct = ql.mem.read(heap_struct_addr, 24)
    printHex(heap_struct)
    _, _, check_addr = struct.unpack('QQQ', heap_struct)
    ql.mem.write(check_addr, b'\x01')

def search_mem_to_find_struct(ql: Qiling):
    MAGIC = ql.pack64(0x3DFCD6EA00000539)
    candidate_addrs = ql.mem.search(MAGIC)

    for addr in candidate_addrs:
        # 有可能有多个地址，所以通过其他特征进一步确认
        stru_addr = addr - 8
        stru = ql.mem.read(stru_addr, 24)
        string_addr, _, check_addr = struct.unpack('QQQ', stru)
        if ql.mem.string(string_addr) == 'Random data':
            ql.mem.write(check_addr, b'\x01')
            break

def challenge8(ql: Qiling):
    base = ql.mem.get_lib_base(ql.path)
    # method 1
    # ql.hook_address(hook_struct, base + 0xFB5)
    # method 2
    ql.hook_address(search_mem_to_find_struct, base + 0xFB5)

```




## challenge 9: 修改字符串函数

```c
unsigned __int64 __fastcall challenge9(bool *a1)
{
  char *i; // [rsp+18h] [rbp-58h]
  char dest[32]; // [rsp+20h] [rbp-50h] BYREF
  char src[40]; // [rsp+40h] [rbp-30h] BYREF
  unsigned __int64 v5; // [rsp+68h] [rbp-8h]

  v5 = __readfsqword(0x28u);
  strcpy(src, "aBcdeFghiJKlMnopqRstuVWxYz");
  strcpy(dest, src);
  for ( i = dest; *i; ++i )
    *i = tolower(*i);
  *a1 = strcmp(src, dest) == 0;
  return __readfsqword(0x28u) ^ v5;
}
```


要么让`tolower()`失效，要么让`strcmp()`失效。

### 脚本

```python
def fake_tolower(ql: Qiling):
    return

def challenge9(ql: Qiling):
    ql.set_api('tolower', fake_tolower)
```




## challenge10: 劫持文件系统，返回指定命令行

```c
unsigned __int64 __fastcall challenge10(_BYTE *a1)
{
  int i; // [rsp+10h] [rbp-60h]
  int fd; // [rsp+14h] [rbp-5Ch]
  ssize_t v4; // [rsp+18h] [rbp-58h]
  char buf[72]; // [rsp+20h] [rbp-50h] BYREF
  unsigned __int64 v6; // [rsp+68h] [rbp-8h]

  v6 = __readfsqword(0x28u);
  fd = open("/proc/self/cmdline", 0);
  if ( fd != -1 )
  {
    v4 = read(fd, buf, 0x3FuLL);
    if ( v4 > 0 )
    {
      close(fd);
      for ( i = 0; v4 > i; ++i )
      {
        if ( !buf[i] )
          buf[i] = 32;
      }
      buf[v4] = 0;
      if ( !strcmp(buf, "qilinglab") )
        *a1 = 1;
    }
  }
  return __readfsqword(0x28u) ^ v6;
}
```


让我们能从`/proc/self/cmdline`读到`"qilinglab"`，故技重施即可，hook文件系统。

### 脚本

```python
class Fake_cmdline(QlFsMappedObject):
    def read(self, expected_len):
        return b'qilinglab'
    
    def close(self):
        return 0

def challenge10(ql: Qiling):
    ql.add_fs_mapper('/proc/self/cmdline', Fake_cmdline())
```




### 简便方法

可以直接替换指定文件成我们的文件，先创建我们需要的文件：

```bash
echo -n "qilinglab" > fake_cmdline

```


然后代码就可以这样写了：

```python
def challenge10(ql):
    ql.add_fs_mapper("/proc/self/cmdline", "./fake_cmdline")
```




还一种方法，代码也不用写了，直接往qiling的rootfs里面写：

```bash
mkdir -p ./qiling/examples/rootfs/x8664_linux/proc/self
echo -n "qilinglab" > ./qiling/examples/rootfs/x8664_linux/proc/self/cmdline
```




## challenge 11: 指令hook

这里指的是用qiling的指令`hook_code()`而不是`ql.hook_insn()`，先看下目标

```c
unsigned __int64 __fastcall challenge11(_BYTE *a1)
{
  int v7; // [rsp+1Ch] [rbp-34h]
  int v8; // [rsp+24h] [rbp-2Ch]
  char s[4]; // [rsp+2Bh] [rbp-25h] BYREF
  char v10[4]; // [rsp+2Fh] [rbp-21h] BYREF
  char v11[4]; // [rsp+33h] [rbp-1Dh] BYREF
  unsigned __int64 v12; // [rsp+38h] [rbp-18h]

  v12 = __readfsqword(0x28u);
  _RAX = 0x40000000LL;
  __asm { cpuid }
  v7 = _RCX;
  v8 = _RDX;
  if ( __PAIR64__(_RBX, _RCX) == 0x696C6951614C676ELL && (_DWORD)_RDX == 538976354 )
    *a1 = 1;
  // ...
}
```


看下汇编：

```NASM
cpuid
mov     eax, edx
mov     esi, ebx
mov     [rbp+var_30], esi
mov     [rbp+var_34], ecx
mov     [rbp+var_2C], eax
cmp     [rbp+var_30], 696C6951h
jnz     short loc_564846E011C0
cmp     [rbp+var_34], 614C676Eh
jnz     short loc_564846E011C0
cmp     [rbp+var_2C], 20202062h
jnz     short loc_564846E011C0
```


目标就很明确了。cpuid指令会填充几个寄存器，具体可以参考intel手册。

### 脚本

```python
def hook_cpuid(ql: Qiling, address, size):
    """
    0000564846E0118F 0F A2      cpuid
    """
    if ql.mem.read(address, size) == b'\x0F\xA2':
        ql.reg.ebx = 0x696C6951
        ql.reg.ecx = 0x614C676E
        ql.reg.edx = 0x20202062
        ql.reg.rip += 2


def challenge11(ql: Qiling):
    begin, end = 0, 0
    for info in ql.mem.map_info:
        if info[2] == 5 and info[3] == '/mnt/d/Playground/QilingLab/qilinglab-x86_64':
            begin, end = info[:2]

    ql.hook_code(hook_cpuid, begin=begin, end=end)
```


说明：

`ql.mem.map_info`也就是`ql.mem.show_mapinfo()`的内容，5表示的是`r-x`属性，加这个判断也是为了缩小hook的范围，提高性能。



## 总结

好耶！

```bash
Challenge 1: SOLVED
Challenge 2: SOLVED
Challenge 3: SOLVED
Challenge 4: SOLVED
Challenge 5: SOLVED
Challenge 6: SOLVED
Challenge 7: SOLVED
Challenge 8: SOLVED
Challenge 9: SOLVED
Challenge 10: SOLVED
Challenge 11: SOLVED
You solved 11/11 of the challenges
```


到此位置这个练习就全部结束了！感觉也没啥太多坑的点，有的话根据报错信息来解决就好。
