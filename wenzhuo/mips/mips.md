

```asm
Decompilation of print.s:

main: 
0x00400000 [0x20020004]    addi   $v0, $zero, 4
0x00400004 [0x3c041001]    lui    $a0, 4097
0x00400008 [0x34840000]    ori    $a0, $a0, 0
0x0040000c [0x0000000c]    syscall 
0x00400010 [0x2002000a]    addi   $v0, $zero, 10
0x00400014 [0x0000000c]    syscall
```

![alt text](image.png)

```asm
.data
    # 定义要打印的字符串
    message: .asciiz "Well, this was a MIPStake!\n"

.text
    # 主程序入口
    .globl main

main:
    # 将 $a0 寄存器设置为 1，因为 printf 返回值通过 $a0 返回
    li $v0, 4          # 加载系统调用号 4 到 $v0（print_string）
    la $a0, message    # 将字符串地址加载到 $a0
    syscall            # 调用系统调用打印字符串

    # 返回 0
    li $v0, 10         # 加载系统调用号 10 到 $v0（exit）
    syscall            # 调用系统调用退出程序
```