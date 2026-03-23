## 函数的调用以及栈帧的利用
调用函数的本质其实是通过改变下一步要执行程序的地址来进行操作的

因此我们存在两种不同的跳转语句：
```bash
jal ra label //简写为jal label(这种时候不允许多个ra省略的套用)
jalr ra, rs1, offset
```
这两种的作用都是跳转，但是含义相同，第一个`jal`是跳转到特定的`target = PC+(label<<1)`地址，但是将本函数后`PC+4`的地址存入`ra`中,而第二种方式是跳转到`target = (rs1 + offset) &~ 1`的位置，然后也将这个`PC + 4`的地址存在这个`ra`中。

因此我们常见的做法是`jal ra func1`,`jalr x0 ra 0`这两个共同使用，就是一个基本的函数的调用和返回的操作

但是往往在实际的调用和实现中，我们往往会发现寄存器无法全部存下我们的数据，因此我们需要利用栈帧进行数据传递，栈帧的实现原理如下：

```c
#include <stdio.h>
int sum_two_number(int a, int b){
    int y;
    return y=a+b;
}
int main(int argc, const char * argv[]) {
    int x=4321, y=1234;
    int a=1,b=2,c=3,d=4,e=5,f=6,g=0;
    y = sum_two_number(x,y);
    c = sum_two_number(a,b);
    f = sum_two_number(e,d);
    g = sum_two_number(c,f);
    printf("Sum is %d.\n",y);
    return 0;
}
```
1. 首先进入`int main()`函数，第一行代码是`int x = 4321, y = 1234;`那么这个时候会将这两个参数首先存在这个寄存器中`t0 = 4321,t1 = 1234`，然后立马转移到栈帧中`sw t0, 0(s0)`,`sw t0, 4(s0)`

2. 我们再来看这个`int a=1,b=2,c=3,d=4,e=5,f=6,g=0;`这个式子再真正的处理的时候和上面的方式相同，也是先将这个保存在寄存器中，然后转移到栈帧中`sw t0, 8(s0)`,`sw t0, 12(s0)`,`sw t0, 16(s0)`,`sw t0, 20(s0)`,`sw t0, 24(s0)`,`sw t0, 28(s0)`,`sw t0, 32(s0)`

3. 函数的调用的过程，在这个过程中，我们首先要从`main`的栈中将`x,y`加载到这个寄存器中,首先在这个`int y`的过程中，我们为它开辟一个地址(会计算这个需要的地址，比如这里存在一个变量`y`，以及一个地址`ra`,这里的a,b可以采用临时的地址在寄存器中读取)，然后`return y = a+b`，我们存储这个结果，然后`jalr x0 ra 0`
```bash
simple:
    addi sp, sp, -16    # 编译器分配的固定大小
    sw ra, 12(sp)       # 保存 ra
    add t0, a0, a1      # c = a + b
    mv a0, t0           # 返回值
    lw ra, 12(sp)       # 恢复 ra
    addi sp, sp, 16     # 释放
    ret
```

那么综上，上述这个例子的一个完整的汇编语言如下：

```assembly
sum_two_number:
    addi sp, sp, -16        # 分配16字节栈空间
    sw ra, 12(sp)           # 保存返回地址

    add t0, a0, a1          # t0 = a + b
    sw t0, 8(sp)            # 存到局部变量 y (可选)
    mv a0, t0               # 返回值放到 a0

    lw ra, 12(sp)           # 恢复返回地址
    addi sp, sp, 16         # 释放栈帧
    jr ra                   # 返回 (或 ret)

main:
    
    addi sp, sp, -64        # 分配64字节栈空间（最低地址，表示栈顶）
    sw ra, 60(sp)           # 保存返回地址 (返回到启动代码)
    sw s0, 56(sp)           # 保存帧指针(除去ra之后的最高地址，表示栈底，在整个函数中稳定)
    addi s0, sp, 64         # 设置帧指针
        
    # int x = 4321, y = 1234;
    li t0, 4321             # t0 = 4321
    sw t0, -48(s0)          # x 存在 [s0-48] 的位置
    li t0, 1234             # t0 = 1234
    sw t0, -44(s0)          # y 存在 [s0-44] 的位置
    
    # int a = 1, b = 2, c = 3, d = 4, e = 5, f = 6, g = 0;
    li t0, 1                # a = 1
    sw t0, -40(s0)          # a 存在 [s0-40]
    li t0, 2                # b = 2
    sw t0, -36(s0)          # b 存在 [s0-36]
    li t0, 3                # c = 3
    sw t0, -32(s0)          # c 存在 [s0-32]
    li t0, 4                # d = 4
    sw t0, -28(s0)          # d 存在 [s0-28]
    li t0, 5                # e = 5
    sw t0, -24(s0)          # e 存在 [s0-24]
    li t0, 6                # f = 6
    sw t0, -20(s0)          # f 存在 [s0-20]
    li t0, 0                # g = 0
    sw t0, -16(s0)          # g 存在 [s0-16]
    
    # 3. y = sum_two_number(x, y);
    lw a0, -48(s0)          # 从栈加载 x 到 a0
    lw a1, -44(s0)          # 从栈加载 y 到 a1
    jal ra, sum_two_number  # 调用 sum_two_number
    sw a0, -44(s0)          # 返回值存回栈中的 y
    ...

    # 恢复寄存器并释放栈帧
    lw ra, 60(sp)           # 恢复返回地址
    lw s0, 56(sp)           # 恢复帧指针
    addi sp, sp, 64         # 释放栈帧
    jr ra                   # 返回
```

总结一下：函数调用的流程
1. `caller`准备，将参数放入`a0~a7`中，必要的时候需要保存在`caller-saved`中
2. 用`jal`跳转到`callee`,同时采用`ra`记录地址（如果是叶子函数（不会调用其他函数）那么不需要额外记录`ra`到内存，其他的就还是需要将这个`ra`记录到内存）
3. `callee`准备，调整`sp`（不需要储存在栈中，`s0`注意不要动），同时需要将`callee-saved`的内容传入到函数中
4. 执行函数逻辑
5. `sp`回到原来位置，同时将存在栈中的`ra`传入到寄存器中
6. `(ret)`,`jr ra`

## 常见的RISC-V指令以及寄存器
- 一般而言，我们在自己写寄存器的指令的时候会使用到的寄存器是`x0`,`t0~t6`,`a0~a7`,`s0~s11`,`sp`,`ra`.
- 这一些寄存器的使用有严格的要求，不能够随便使用，一般而言`t0~t6`是临时寄存器，我们可以临时使用，不用做任何处理，`a0~a7`需要我们按照一定规则使用，例如`a7`的不同值的时候调用`ecall`的结果完全不同，`a7 = 1`会打印`a0`储存的值，`a7 = 4`则会打印这个`a0`储存的字符串的地址的字符串。`a7 = 5`,`a0`则会储存输入的结果，`a7 = 10`则会退出程序。

- `sp`,`ra`的使用往往和函数的调用有关，在上述的函数调用的汇编语言的书写中已经详细写清楚了这个逻辑。
### 常见指令——运算
```assembly
add rd, rs1, rs2
sub rd, rs1, rs2
mul rd, rs1, rs2
div rd, rs1, rs2
and rd, rs1, rs2
or rd, rs1, rs2
xor rd, rs1, rs2
```
### 常见指令——比较
```assembly
slt  rd, rs1, rs2   # 有符号数比较
sltu rd, rs1, rs2   # 无符号数比较
```
### 常见指令——移位
```assembly
sll rd, rs1, rs2  # Shift Left Logical 逻辑左移
srl rd, rs1, rs2  # Shift Right Logical 逻辑右移（最高位补0，适用于无符号数）
sra rd, rs1, rs2  # Shift Right Arithmetic 算术右移（最高位补符号位，适用于有符号数）
```
### 常见指令——分支指令
```assembly
beq rs1, rs2, label # 比较两个寄存器是否相等
bne rs1, rs2, label # 比较两个寄存器是否不相等
blt rs1, rs2, label # if rs1 < rs2
bge rs1, rs2, label # if rs1 >= rs2
bltu rs1, rs2, label # if rs1 < rs2
bgeu rs1, rs2, label # if rs1 >= rs2
jal rd, label # 跳转到label，并把返回地址保存在rd中
jalr rd, rs1, imm # 跳转到rs1 + imm地址处的指令，并把返回地址保存在rd中
jal offset = jal ra offset
j offset = jal x0 offset
```

### 常见指令——地址与内存的处理
```assembly
s0: word. 9
.text
lw t0, 0(s0)
sw t0, 0(s0)
```
- `lw t0, 0(s0)`：从`s0`地址处加载一个4字节的数据到`t0`中，这个指令会自动将地址转换为物理地址，然后从物理地址中加载数据到`t0`中。
- `sw t0, 0(s0)`：将`t0`中的数据写入到`s0`地址处，这个指令会自动将地址转换为物理地址，然后将数据写入到物理地址中。
**需要说明的是**
1. 在这个地方中的0是一个立即数，表示的是地址的偏移量，这个偏移量会自动加上`s0`的地址，然后转换为物理地址，然后从物理地址中加载数据到`t0`中。
2. 这个过程中，其实包含着两个语句`auipc t0, %pcrel_hi(s0)`和`lw t0, %pcrel_lo(n)(t0)`,其中`%pcrel_hi(s0)`是`(s0的地址-PC)`的高20位，`%pcrel_lo(n)`是`(s0的地址-PC)`的低12位。