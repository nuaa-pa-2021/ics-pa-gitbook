# mov指令执行例子剖析

在PA1中, 你已经阅读了 monitor 部分的框架代码, 了解了 NEMU 执行的粗略框架. 但现在, 你需要进一步弄明白, 一条指令是怎么在 NEMU 中执行的, 即我们需要进一步探究 `exec_wrapper()` 函数中的细节. 为了说明这个过程, 我们举了两个 `mov` 指令的例子, 它们是 NEMU 自带的客户程序 `mov` 中的两条指令:

```
100000:    b8 34 12 00 00          mov    $0x1234,%eax
......
100017:    66 c7 84 99 00 e0 ff    movw   $0x1,-0x2000(%ecx,%ebx,4)
10001e:    ff 01 00
```

## 简单 mov 指令的执行

对于大部分指令来说, 执行它们都可以抽象成`取指—译码—执行`的[指令周期](http://en.wikipedia.org/wiki/Instruction_cycle). 为了使描述更加清晰, 我们借助指令周期中的一些概念来说明指令执行的过程. 我们先来剖析第一条 `mov $0x1234, %eax` 指令的执行过程.

### 取指(instruction fetch, IF)

要执行一条指令, 首先要拿到这条指令. 指令究竟在哪里呢? 还记得冯诺依曼体系结构的核心思想吗? 那就是"存储程序, 程序控制". 你以前听说这两句话的时候可能没有什么概念, 现在是实践的时候了. 这两句话告诉你, 指令在存储器中, 由 PC(program counter, 在x86中就是 `eip`) 指出当前指令的位置. 事实上, `eip`就是一个指针!  取指令要做的事情自然就是将 `eip` 指向的指令从内存读入到 CPU 中. 在 NEMU 中, 有一个函数 `instr_fetch()`(在`nemu/include/cpu/exec.h` 中定义)专门负责取指令的工作.

### 译码(instruction decode, ID)

在取指阶段, CPU 拿到的是指令的比特串. 如果想知道这串比特串究竟代表什么意思, 就要进行译码的工作了. 我们可以把译码的工作作进一步的细化: 首先要决定具体是哪一条指令的哪一种形式, 这主要是通过查看指令的**操作码**（`opcode`）来决定的. 对于大多数指令来说, CPU只要看指令的第一个字节就可以知道具体指令的形式了. 在NEMU中, `exec_real()`（实际代码中形式是`make_Ehelper(real)`）函数首先通过 `instr_fetch()` 取出指令的第一个字节, 将其解释成 `opcode` 并记录在全局译码信息 `decoding` 中. 然后通过 `set_width()` 函数(在 `nemu/src/cpu/exec/exec.c` 中定义)记录默认的操作数宽度. 若操作数宽度结果为 `0`, 表示光看操作码的首字节, 操作数宽度还不能确定, **可能是16位或者32位**, 需要通过 `decoding.is_operand_size_16` 成员变量来决定. 这其实实现了"操作数宽度前缀"的相关功能, 关于`is_operand_size_16`成员的更多内容会在下文进行说明.

返回后, `exec_real()` 接下来会根据取到的 `opcode` 查看 `opcode_table`, 得到指令的译码 helper 函数和执行helper 函数, 并将其作为参数调用 `idex()` 函数来继续模拟这条指令的执行. `idex()` 函数的原型为

```c
void idex(vaddr_t *eip, opcode_entry *e);
```

它的作用是通过 `e->decode` 函数(若不为`NULL`)对参数 `eip` 指向的指令进行译码, 然后通过 `e->execute` 函数执行这条指令.

以 `mov $0x1234, %eax` 指令为例, 首先通过 `instr_fetch()` 取得这条指令的第一个字节 `0xb8`, 然后将这个字节作为 opcode 来索引 `opcode_table`, 发现这一指令的操作数宽度是 `4` 字节, 并通过 `set_width()` 函数记录. 接着按照同样的方式来索引 `opcode_table`，这样就确定取到的是一条 `mov` 指令, 它的形式是将立即数移入寄存器(move immediate to register).

{% panel style="info", title="指令太多怎么办？" %}

事实上, 一个字节最多只能区分 256 种不同的指令形式. 当指令形式的数目大于 256 时, 我们需要使用另外的方法来识别它们. x86 中有主要有两种方法来解决这个问题 (在 PA2 中你都会遇到这两种情况):

- 一种方法是使用转义码(escape code), x86 中有一个 **2 字节转义码** `0x0f`, 当指令`opcode`的第一个字节是`0x0f`时, 表示需要再读入一个字节才能决定具体的指令形式(部分条件跳转指令就属于这种情况). 后来随着各种SSE指令集的加入, 使用2字节转义码也不足以表示所有的指令形式了, x86在2字节转义码的基础上又引入了3字节转义码, 当指令`opcode`的前两个字节是`0x0f`和`0x38`时, 表示需要再读入一个字节才能决定具体的指令形式.
- 另一种方法是使用 `ModR/M` 字节中的扩展 opcode 域来对 `opcode` 的长度进行扩充. 有些时候, 读入一个字节也还不能完全确定具体的指令形式, 这时候需要读入紧跟在 `opcode` 后面的 `ModR/M` 字节, 把其中的`reg/opcode` 域当做 `opcode` 的一部分来解释, 才能决定具体的指令形式. x86 把这些指令划分成不同的指令组(instruction group), 在同一个指令组中的指令需要通过 `ModR/M` 字节中的扩展 opcode 域来区分.

{% endpanel %}

决定了具体的指令形式之后, 译码工作还需要决定指令的**操作数**. 事实上, 在确定了指令的 `opcode` 之后, 指令形式就能确定下来了, CPU 可以根据指令形式来确定具体的操作数. 对于 `mov $0x1234, %eax` 指令来说, 确定操作数其实就是确定寄存器 `%eax` 和立即数 `$0x1234`. 在 x86 中, 通用寄存器都有自己的编号,`I2r` 形式的指令把寄存器编号也放在指令的第一个字节里面, 我们可以通过位运算将寄存器编号抽取出来; 立即数存放在指令的第二到第五个字节, 可以很容易得到它. 需要说明的是, 由于立即数是指令的一部分, 我们还是通过 `instr_fetch()` 函数来获得它. 总的来说, 由于指令变长的特性, 指令长度和指令形式需要一边取指一边译码来确定, 而不像 RISC 指令集那样可以泾渭分明地处理取指和译码阶段, 因此你会在 NEMU 的实现中看到译码函数里面也会有 `instr_fetch()` 的操作.

### 执行(execute, EX)

译码阶段的工作完成之后, CPU就知道当前指令具体要做什么了, 执行阶段就是真正完成指令的工作. 对于 `mov $0x1234, %eax` 指令来说, 执行阶段的工作就是把立即数 `$0x1234` 送到寄存器 `eax` 中. 由于 `mov` 指令的功能可以统一成"把源操作数的值传送到目标操作数中", 而译码阶段已经把操作数都准备好了, 所以只需要针对 `mov` 指令编写一个模拟执行过程的函数即可. 这个函数就是 `exec_mov()`, 它是通过 `make_EHelper` 宏来定义的，该函数存放在 `nemu/src/cpu/exec/data_move.c` 目录下:

```c
make_EHelper(mov) {
    write_operand((id_dest, &id_src->val);
    print_asm_template2(mov);
}
```

其中 `write_operand()` 函数会根据第一个参数中记录的类型的不同进行相应的写操作, 包括写寄存器和写内存. `print_asm_template2()` 是个宏, 用于输出带有两个操作数的指令的汇编形式.

### 更新 `eip`

执行完一条指令之后, CPU 就要执行下一条指令. 在这之前, CPU 需要更新 `eip` 的值, 让 `eip` 指向下一条指令的位置. 为此, 我们需要确定刚刚执行完的指令的长度. 事实上, 在 `instr_fetch()` 中, 每次取指都会更新它的 `eip` 参数, 而这个参数就是在 `exec_wrapper()` 调用 `exec_real()` 时传入的 `decoding.seq_eip`. 因此当`exec_wrapper()` 执行完一条指令调用 `update_eip()` 时, `decoding.seq_eip` 已经正确指向下一条指令了, 这时候直接更新 `eip` 即可.

## 复杂 mov 指令的执行

对于第二个例子 `movw $0x1, -0x2000(%ecx,%ebx,4)`, 执行这条执行还是分取指, 译码, 执行三个阶段.

首先是取指. 这条 mov 指令比较特殊, 它的第一个字节是 `0x66`, 如果你查阅 i386 手册, 你会发现 `0x66` 是一个`operand-size prefix`. 因为这个前缀的存在, 本例中的`mov`指令才能被CPU识别成`movw`. NEMU使用`decoding.is_operand_size_16` 成员变量来记录操作数宽度前缀是否出现, `0x66` 的 helper 函数`operand_size()` 实现了这个功能. `operand_size()` 函数对 `decoding.is_operand_size_16` 成员变量做了标识之后, 越过前缀重新调用 `exec_real()` 函数, 此时取得了真正的操作码 `0xc7`. 由于`decoding.is_operand_size_16` 成员变量进行过标识, 在 `set_width()` 函数中将会确定操作数长度为 `2` 字节.

接下来是识别操作数. 根据操作码 `0xc7` 查看 `opcode_table`, 调用译码函数 `decode_mov_I2E()`, 这个译码函数又分别调用 `decode_op_I()` 和 `decode_op_rm()` 来取出操作数. 阅读代码, 你会发现 `decode_op_rm()` 最终会调用 `read_ModR_M()` 函数. 由于本例中的 `mov` 指令需要访问内存, 因此除了要识别出立即数之外, 还需要确定好要访问的内存地址. x86 通过 `ModR/M` 字节来指示内存操作数, 支持各种灵活的寻址方式. 其中最一般的寻址格式是

```
displacement(R[base_reg], R[index_reg], scale_factor)
```

相应内存地址的计算方式为

```
addr = R[base_reg] + R[index_reg] * scale_factor + displacement
```

其它寻址格式都可以看作这种一般格式的特例, 例如

```
displacement(R[base_reg])
```

可以认为是在一般格式中取 `R[index_reg] = 0, scale_factor = 1` 的情况. 这样, 确定内存地址就是要确定`base_reg`,  `index_reg`,  `scale_factor` 和 `displacement` 这 4 个值, 而它们的信息已经全部编码在 `ModR/M` 字节里面了.

我们以本例中的 `movw $0x1, -0x2000(%ecx,%ebx,4)` 说明如何识别出内存地址:

```
100017:    66 c7 84 99 00 e0 ff    movw   $0x1,-0x2000(%ecx,%ebx,4)
10001e:    ff 01 00
```

根据 `mov_I2E` 的指令形式, `0xc7` 是 `opcode`, `0x84` 是 `ModR/M` 字节. 在 i386 手册中查阅表格 17-3 得知, `0x84` 的编码表示在 `ModR/M` 字节后面还跟着一个 `SIB` 字节, 然后跟着一个32位的 `displacement`. 于是读出 `SIB` 字节, 发现是 `0x99`. 在 i386 手册中查阅表格 17-4 得知, `0x99` 的编码表示 `base_reg = ECX, index_reg = EBX, scale_factor = 4`. 在 `SIB` 字节后面读出一个32位的 `displacement`, 发现是 `00 e0 ff ff`, 在小端存储方式下, 它被解释成 `-0x2000`. 于是内存地址的计算方式为

```
addr = R[ECX] + R[EBX] * 4 - 0x2000
```

框架代码已经实现了 `load_addr()` 函数和 `read_ModR_M()` 函数(在 `nemu/src/cpu/decode/modrm.c` 中定义), 它们的函数原型为

```c
void load_addr(swaddr_t *eip, ModR_M *m, Operand *rm);
void read_ModR_M(swaddr_t *eip, Operand *rm, bool load_rm_val, Operand *reg, bool load_reg_val);
```

它们将变量 `eip` 所指向的内存位置解释成 `ModR/M` 字节, 根据上述方法对 `ModR/M` 字节和 `SIB` 字节进行译码, 把译码结果存放到参数 `rm` 和 `reg` 指向的变量中. 虽然 i386 手册中的表格 17-3 和表格 17-4 内容比较多, 仔细看会发现, `ModR/M` 字节和 `SIB` 字节的编码都是有规律可循的, 所以 `load_addr()` 函数可以很简单地识别出计算内存地址所需要的 4 个要素(当然也处理了一些特殊情况). 不过你现在可以不必关心其中的细节, 框架代码已经为你封装好这些细节, 并且提供了各种用于译码的接口函数.

本例中的执行阶段就是要把立即数写入到相应的内存位置. 译码阶段已经把操作数准备好了, 执行函数 `exec_mov()` 会完成数据移动的操作, 最终在 `update_eip()` 函数中更新 `eip`.