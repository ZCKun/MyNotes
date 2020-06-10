# ELF ARM - Basic Crackme


## 说明
> 这是一个arm32的程序，[ELF ARM - Basic Crackme](https://www.root-me.org/en/Challenges/Cracking/ELF-ARM-basic-crackme)

----
## 分析
看向流程图，第一个块我一开始以为没什么好看的，后来越发觉得这里面做了很多工作，忽略这个块就会搞得你看后面的一头雾水。

就是获取argc的长度，看是不是小于2，小于2说明没有输入password，直接puts信息提示输入。

![](/Volumes/HDD500G/Documents/CTF/root-me/ELF_ARM-Basic_Crackme/graph_first_block.png)

一行一行的分析，这里我会用到画图的方式来更好的理解堆栈

> 1、首先是往栈中放些东西，注意这里用的不是push，因为push指令只在thumb模式中有，而这个程序是arm模式的；r11是栈帧指针，lr是链接寄存器，具体可以看我之前的文章 [ARM数据类型](http://www.2h0n9.com/2019/11/03/arm-data-type-191103)；这条指令首先会减少sp的值，也就是扩大堆栈的空间，然后将r4,r11,r14寄存器以`从右往左的顺序`放到分配好的栈里(LR -> R11(FP) -> R4)
> ```arm
> STMFD   SP!, {R4,R11,LR}
> ```
> 堆栈情况如图
> ![](/Volumes/HDD500G/Documents/CTF/root-me/ELF_ARM-Basic_Crackme/stack_begin.png)
> 2、这条指令就是改变了下fp的位置，即指向刚刚放入堆栈的LR寄存器(`记住一个寄存器32位，也就是32/8=4个字节`)
> ```arm
> ADD     R11, SP, #8
> ```
> ![](/Volumes/HDD500G/Documents/CTF/root-me/ELF_ARM-Basic_Crackme/stack_chage_fp_addr.png)
> 3、我们都知道arm有个函数调用约定 ([ARM数据类型](http://www.2h0n9.com/2019/11/03/arm-data-type-191103/))，就是r0-r3用作函数的前4个参数，这里用到的是r0和r1；在arm中，有30个32位的通用寄存器，这意味着每个寄存器的大小都是4个字节；
这条指令的主要作用是分配28个字节的栈空间来存放局部变量
> ```arm
> SUB     SP, SP, #0x1C
> ```
> 所以此时的堆栈应该是这样的
> ![](/Volumes/HDD500G/Documents/CTF/root-me/ELF_ARM-Basic_Crackme/stack_chage_sp_save_local_value.png)
> 4、分配了栈空间，那肯定要放东西了；这条指令就是在 `r11 + (-0x20)` 也就是 `0xbefff13c + (-0x20) = 0xbefff11c` 处存放参数1，这个地址是在sp指针的下面那个栈
> ```arm
> STR     R0, [R11,#var_20]
> ```
> (因为这里改变不大，我就懒得上堆栈图了，结合下面的几条指令一起上图)
> 这条指令是把R1(参数2)存放到 `r11 + (-0x24) = 0xbefff118` 也就是栈顶SP处
> ```arm
> STR     R1, [R11,#var_24]
> ```
> 然后分别存放0x6、0x0到0xbefff12c、0xbefff128处
> ```arm
> MOV     R3, #6
> STR     R3, [R11,#var_10]
> MOV     R3, #0
> STR     R3, [R11,#var_14]
> ```
> 假设我们什么也没输入，此时R0=1，如果要问为什么等于1，那你需要去补充下C语言的知识，因为main方法的第一个参数是第二参数的长度，第二个参数是从命令行接收的用户输入；结合上面的多条指令，所以此时堆栈应该长这样
> ![](/Volumes/HDD500G/Documents/CTF/root-me/ELF_ARM-Basic_Crackme/stack_any_01.png)
> 5、接着就是取出参数1，然后比较是否等于2，如果等于就跳转到0x84B0，否则就输入loser...，然后exit 并传个1
> ```arm
> LDR     R3, [R11,#var_20]
> CMP     R3, #2
> BEQ     loc_84B0
> ```

----

直接看true执行的块

![](/Volumes/HDD500G/Documents/CTF/root-me/ELF_ARM-Basic_Crackme/graph_two_block.png)

来一行一行分析
> 首先是从`fp+0x24`处获取数据到r3寄存器，这个地址存的是参数2，自己看上面的代码就知道了
> ```arm
> LDR     R3, [R11,#var_24]
> ```
> 然后是从`fp+0x24+0x4`处获取数据到r3寄存器，这个地址是我们在命令行运行该程序输入的内容；因为r3的地址就是参数2，再因为这是32位的程序，char*的大小为4个字节，所以偏移量是+0x4
> ```arm
> LDR     R3, [R3,#4]
> ```
> 接着看，这行的意思是把r3又存到 `fp+0x18` 地址处
> ```arm
> STR     R3, [R11,#s]
> ```
> 然后是加载格式化字符串，放在r3寄存器
> ```arm
> LDR     R3, =aCheckingSForPa
> ```
> 然后又移动到r0寄存器来用作printf的第一个参数
> ```arm
> MOV     R0, R3
> ```
> 接着取出输入的内容，放在r1寄存器用作printf的第二参数
> ```arm
> LDR     R1, [R11,#s]
> ```
> 然后就是调用printf
> ```
> BL      printf
> ```
> 接着又从 `fp+0x18` 取出输入内容，因为刚刚调用了printf
> ```arm
> LDR     R0, [R11,#s]
> ```
> 然后调用strlen，参数是r0，获取输入内容的长度并把长度放在r3寄存器，记住r0在每个方法执行结束后就是该方法的返回值
> ```arm
> BL      strlen
> MOV     R3, R0
> ```
> 然后把长度r3存到 `fp+0x1c` 地址，接着又取出来放在r3寄存器。。。
> ```arm
> STR     R3, [R11,#status]
> LDR     R3, [R11,#status]
> ```
> 然后判断长度是否小于6，如果小于就跳转执行puts输出"Loser..." (受到了作者的嘲讽
> ```arm
> CMP     R3, #6
> BEQ     loc_84F8
> ```

接着看 fasle 块

![](/Volumes/HDD500G/Documents/CTF/root-me/ELF_ARM-Basic_Crackme/graph_three_block.png)

还是一行一行分析
> 我们知道r11至始至终都没变过，所以这里还是取 `0xbefff13c+(-0x10)=0xbefff12c` 处的数据，这个地址在第一个块也就是我们最开始分析的时候存过一个数据，那就是0x6，方便更好的理解，我把上面的堆栈图再上一次
> ![](/Volumes/HDD500G/Documents/CTF/root-me/ELF_ARM-Basic_Crackme/stack_any_01.png)
> ```arm
> LDR     R4, [R11,#var_10]
> ```
> 然后又是获取输入的字符串，并获取它的长度放在R3
> ```arm
> LDR     R0, [R11,#s] 
> BL      strlen
> MOV     R3, R0
> ```
> 紧接着这里有个反转减法运算操作，反转减法就是把两个要算的数调换位置算，比如这条指令就是 `r3 = r4 - r3`，这里开始我们需要仔细分析，因为这里到了算法部分，这的反转减法是用 `6 - 字符串长度`
> ```arm
> RSB     R3, R3, R4
> ```
> 接着把运算结果存到刚刚取出0x6的地址
> ```arm
> STR     R3, [R11,#var_10]
> ```
> 然后又把输入的字符串取出来，我先在这说明一下，这里 `r11+#s` 的地址的数据还是一个地址，在这个地址中的数据才是输入的字符串，我也觉得绕，所以我有个办法能帮助我们更好理解
> ```arm
> LDR     R3, [R11,#s]
> ```
>> 首先写个仿照这部分简单写个程序，关于字符串定义啥玩意儿的可以看这里[String definition directives](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0774i/czt1510589813860.html)
>> ```arm
>> .global main               ;程序入口
>> main:                      
>> 0x00000000    MOV		R1,#0xa         ; E3A0100A
>> 0x00000004    MOV		R2,#5           ; E30A2005
>> 0x00000008    ADD		R3,R1,R2        ; E0813002
>> 0x0000000c    LDR		R3,=pass        ; E59F3008
>> 0x00000010    LDRB       R2,[R3]         ; E5D32000
>> 0x00000014    LDR		R3,=pass        ; E51F3000
>> 0x00000018    ADD		R3,R3,#5        ; E2833005
>> 0x0000001c    LDRB       R3, [R3]        ; E5D33000
>>
>> .text
>> 0x00000001c   pass: .asciz "122923"      ; 39323231
>> ```
>> 编译后用十六进制打开差不多是这样，我这里只获取了代码和字符串部分，其他的比如格式啥的就不看了；这里字符串存储用的小端序。
>> ![](/Volumes/HDD500G/Documents/CTF/root-me/ELF_ARM-Basic_Crackme/arm-demo-hex-01.png)
>> 当程序运行，pc会指向0x00000000，也就是第一条指令 `mov r1,#10` 的位置，然后直到0xc处，这里其实写成这样 `ldr r3, [pc, #16]`，就是当pc指向这条指令，那么就直接用pc的地址偏移16个字节，直接到字符串的地址，这里的r3的内容就是0x00000001c；接着就是`ldrb`，这个指令和ldr的最大区别就是它只获取8个字节的数据，比如这里就是获取的r3的地址所指向的字符串的8个字节数据，也就是0x31；然后这里的ldr可以这样写 `ldr r3, [pc,#8]`，接着就是r3在原地址上偏移0x5，然后再用ldrb获取8个字节的数据，那就是最后一个字符 “3” (0x33)

> 回到crackme，接下来是ldrb指令，这里就是获取第一个字符
> ```arm
> LDRB    R2, [R3]
> ```
> 然后是获取最后一个字符
> ```arm
> LDR     R3, [R11,#s]
> ADD     R3, R3, #5
> LDRB    R3, [R3]
> ```
> 接着就是比较，用cmp来使两个字符的ascii码相减，根据结果改变cspr flag，beq是根据z flag来决定是否跳转的，z flag的条件是两个数相减为0时置1，也就是相等就跳转
> ```arm
> CMP     R2, R3
> BEQ     loc_8538
> ```
> 来看看如果相等会怎样；首先从 `r11+(-10)` 地址处取数据，这个地址存的是之前 `6-字符串长度` 的结果，然后自增1，接着存到原始地址；所以这里也就是自增1的作用
> ```arm
> LDR     R3, [R11,#var_10]
> ADD     R3, R3, #1
> STR     R3, [R11,#var_10]
> ```
> 从下图可以看出每个块的最后都有一次比较来看，这里应该是多个if else，然后再自增一个int变量，暂时不清楚这个变量有什么用。
> ![](/Volumes/HDD500G/Documents/CTF/root-me/ELF_ARM-Basic_Crackme/cutter_if_else_blocks.png)
> 直接看下一个判断块
> ```arm
> LDR     R3, [R11,#s]  ;取字符串
> LDRB    R3, [R3]      ;获取第一个字符
> ADD     R2, R3, #1    ;将字符ascii码加1
> LDR     R3, [R11,#s]  ;取字符串
> ADD     R3, R3, #1    ;将字符串地址偏移0x1；str: 0x0  34 33 32 31，r3最开始在0x0，偏移一个字节就到了0x1，也就是33
> LDRB    R3, [R3]      ;从偏移后的地址处取第一个字符
> CMP     R2, R3        ;然后用第一个字符的ascii+1与第二个字符的ascii比较
> BEQ     loc_8564      ;如果相等就自增
> ```
> 然后看第三个判断块
> ```arm
> LDR     R3, [R11,#s]  ;取字符串
> ADD     R3, R3, #3    ;字符串地址偏移3个字节，可以写成r3[3]
> LDRB    R3, [R3]      ;取偏移后的第一个字符
> ADD     R2, R3, #1    ;字符ascii+1
> LDR     R3, [R11,#s]  ;取字符串
> LDRB    R3, [R3]      ;取第一个字符
> CMP     R2, R3        ;用字符的第四个字符ascii与第一个字符ascii比较
> BEQ     loc_8590      ;相等就自增
> ```
> 第四个判断块
> ```arm
> LDR     R3, [R11,#s]  ;取字符串
> ADD     R3, R3, #2    ;将字符串地址便宜2个字节
> LDRB    R3, [R3]      ;取偏移后第一个字符
> ADD     R2, R3, #4    ;字符ascii+4
> LDR     R3, [R11,#s]  ;取字符
> ADD     R3, R3, #5    ;字符串偏移0x5个字节，就到了`输入的`最后一个字符，要知道字符串最后一个字符是`\0`
> LDRB    R3, [R3]      ;取偏移后第一个字符
> CMP     R2, R3        ;用输入的字符串中的第三个字符的ascii+4与输入的字符串中的最后一个字符的ascii比较
> BEQ     loc_85C0      ;相等自增
> ```
> if else块到这就完了，根据上面的逻辑我写了个c代码
> ```c++
> #include <stdio.h>
> #include <string.h>
>
> int main(int args, char **argv)
> {
>     int len = 0;
>     char *pass = argv[1];
>  
>     if (args != 2) {
>         puts("Loser...");
>         exit(1);
>     }
>     printf("Checking %s for password...\n", pass);
>     len = strlen(pass)
>     if (len < tmp) {
>         puts("Loser...");
>         exit(len);
>     }
> 
>     len = 6 - len;
>     if (pass[0] != pass[5]) len++;
>     if (pass[0]+1 != pass[1]) len++;
>     if (pass[3]+1 != pass[0]) len++;
>     if (pass[2]+4 != pass[5]) len++;
>     if (pass[4]+2 != pass[2]) len++;
> 
> }
> ```
> 然后分析最后一个判断块
> ```arm
> LDR     R3, [R11,#s]      ;取字符串
> ADD     R3, R3, #3        ;字符串地址偏移3个字节
> LDRB    R3, [R3]          ;取r3[3]，第四个字符串
> EOR     R3, R3, #0x72     ;逻辑异或，就是二进制相同的为0；这里是第四个字符ascii与0x72逻辑异或运算
> AND     R3, R3, #0xFF     ;逻辑与，和异或相反；这里是将异或后的内容与0xff逻辑与运算
> LDR     R2, [R11,#var_10] ;应该没忘记，这个地址存的是那个自增的数
> ADD     R3, R2, R3        ;自增的数据与运算后的R3相加，结果还是存在r3
> STR     R3, [R11,#var_10] ;把结果存到自增变量的地址
> LDR     R3, [R11,#s]      ;取字符串
> ADD     R3, R3, #6        ;偏移6个字节
> LDRB    R3, [R3]          ;取最后一个字符终止符`\0`
> LDR     R2, [R11,#var_10] ;取运算结果
> ADD     R3, R2, R3        ;相加，运算结果+0
> STR     R3, [R11,#var_10] ;存
> LDR     R3, [R11,#var_10] ;取
> CMP     R3, #0            ;比较
> BNE     loc_8644          ;不相等跳转输出loser，相等输出Success, you rocks
> ```
> 现在可以明确我们需要最后的r3等于0，而r3=与或运算+0，所以我们需要之前的与或运算结果为0；
>> 首先是字符串的第四个字符与0x72('r')异或，然后是与0xff逻辑与
>> 逻辑与的运算是两个数的位为1结果才为1，那么我们就要两个数的没位都为0就行了，而能达成这个条件的的只有数字0；
>> 而这里的逻辑与运算的第二个数0xff我们没法让它变化，所以从逻辑异或中入手
>> 想要异或的结果为0，那么就需要两个数相等，也就是说字符串中的第四个字符串为r
>> 然后后面还有一次运算，这个结果会与之前自增的那个数相加，所以我们还需要让那个自增数为0，也就是那些判断都为false

> 为了方便分析，我更新了下c代码
> ```c++
> #include <stdio.h>
> #include <string.h>
>
> void main(int args, char **argv)
> {
>     int tmp;
>     int len = 0;
>     char *pass = argv[1];
>  
>     if (args != 2) {
>         puts("Loser...");
>         exit(1);
>     }
>     printf("Checking %s for password...\n", pass);
>     len = strlen(pass);
>     if (len < tmp) {
>         puts("Loser...");
>         exit(len);
>     }
> 
>     len = 6 - len;
>     if (pass[0] != pass[5]) len++;
>     if (pass[0]+1 != pass[1]) len++;
>     if (pass[3]+1 != pass[0]) len++;
>     if (pass[2]+4 != pass[5]) len++;
>     if (pass[4]+2 != pass[2]) len++;
>     tmp = ((pass[3]^0x72)&0xFF) + len + pass[6];
>     if (tmp) {
>         puts("Loser...");
>         exit(len);
>     }
>     puts("Success, you rocks!");
>     exit(0);
> }
> ```
> emm...写了两种
> ```c++
> #include <stdio.h>
> #include <string.h>
>
> int main(int args, char **argv)
> {
>     int len = 0;
>     char *pass = argv[1];
>  
>     if (args != 2) {
>         puts("Loser...");
>         exit(1);
>     }
>     printf("Checking %s for password...\n", pass);
>     len = strlen(pass);
>     if (len < 6) {
>         puts("Loser...");
>         exit(len);
>     }
> 
>     len = 6 - len;
>     if (pass[0] == pass[5]) {
>         if (pass[0]+1 == pass[1]) {
>             if (pass[3]+1 == pass[0]) {
>                 if (pass[2]+4 == pass[5]) {
>                     if (pass[4]+2 == pass[2]) {
>                         if (pass[3] == 0x72) {
>                             puts("Success, you rocks!");
>                             exit(len);
>                         }
>                     }
>                 }
>             }
>         }
>     }
>     puts("Loser...");
>     exit(len);
> }
> ```
> 已知password长度为6，第四个字符为0x72，然后来分析判断，初始化 `pass = ['', '', '', '0x72', '', '']`
>> 从已知的地方开始，第四个字符+1等于第一个字符，那么 `pass = ['0x73', '', '', '0x72', '', '']`
>> 接着第一个字符等于第六个字符，`pass = ['0x73', '', '', '0x72', '', '0x73']`
>> 然后第一个字符+1等于第二个字符，`pass = ['0x73', '0x74', '', '0x72', '', '0x73']`
>> 第三个字符+4等于第六个字符，`pass = ['0x73', '0x74', '', '0x72', '0x67', '0x73']`
>> 最后第五个字符+2等于第三个字符，`pass = ['0x73', '0x74', '0x6f', '0x72', '0x6d', '0x73']`

> 所以结果为 `storms`
> ```python
> ''.join([chr(int(i,16)) for i in ['0x73', '0x74', '0x6f', '0x72', '0x6d', '0x73']]) # storms
> ```
> ![](/Volumes/HDD500G/Documents/CTF/root-me/ELF_ARM-Basic_Crackme/result.png)

----

## 最后
期间查的资料以及所用的到网页工具：
[arm_instruction_set_reference_guide_100076_0100_00_en](https://static.docs.arm.com/100076/0100/arm_instruction_set_reference_guide_100076_0100_00_en.pdf)
[ARM数据类型](http://www.2h0n9.com/2019/11/03/arm-data-type-191103/#more)
[armclang_reference_guide_100067_0611_00_en](https://static.docs.arm.com/100067/0611/armclang_reference_guide_100067_0611_00_en.pdf)
[armconverter](http://armconverter.com/)
[onlinedisassembler](https://onlinedisassembler.com/odaweb/)



