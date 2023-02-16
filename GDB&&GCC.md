### GDB trace and debugging

#### 常用指令

next/n: 下一条源码指令（不进入函数调用）
nexti: 下一条机器指令（不进入函数调用）
step/s: 下一条源码指令（进入函数调用）
stepi: 下一条机器指令（进入函数调用）

x 命令： 显示给定内存地址的内容

```shell
x /[Length][Format][bit modify] [Address expression]
[format]
o - octal
x - hexadecimal
d - decimal
u - unsigned decimal
t - binary
f - floating point
a - address
c - char
s - string
i - instruction
[bit modify]
b - byte
h - halfword (16-bit value)
w - word (32-bit value)
g - giant word (64-bit value)

#example:
x /8dw 0x7fffff  # show 8 32-bit decimal integer starting from memory 0x7fffff
```

### GCC extent ASM and built-in funcs
