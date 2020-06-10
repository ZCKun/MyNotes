# 抖音分析篇(二)

打开IDA Pro，载入**libuserinfo.so**

![](/Users/2h0n91i2hen/Pictures/douyin2/idapro_userinfo.png)

在右侧**Function name**窗口按**Ctrl-F**搜索**GetUserInfo**，就是这个**nativeGetUserInfo**

![](/Users/2h0n91i2hen/Pictures/douyin2/idapro_search_function_getuserinfo.png)

双击改方法定位到汇编代码处，然后按F5进行反编译，反编译并不是完整代码，只是能够帮助我们更好的分析逻辑的伪代码，该方法的几个参数我都写了注释

![](/Users/2h0n91i2hen/Pictures/douyin2/idapro_nativegetuserinfo_func_dec.png)

```c++
JNIEnv *env //是个指针，jnienv类型实际上代表了Java环境。
int jobject //如果native方法不是static的话，这个obj就代表这个native方法的类实例。如果native方法是static的话，这个obj就代表这个native方法的类的class对象实例(static方法不需要类实例的，所以就代表这个类的class对象)。
int a3 // 这是serverTime，被我重新命了名
int a4 // 这个是对应java代码中 t.toString()
int a5 // 这个是请求url中参数部分以‘=’为分割符分割后的array
int a6 // 这个是0
```

接着看到这里，我不记得是多少行了，我这里的因为有些内容被我改了所以行数和你们对不上，反正我把伪代码贴上来：
```c++
  if ( params )
    v31 = (const char *)_JNIEnv::GetStringUTFChars(env, a4, 0);
  else
    v31 = (const char *)&unk_598B4;
```
这里是一个将jstring转chars的操作，熟悉GetStringUTFChars是个JNI方法，它长这样：
```
const char * GetStringUTFChars(JNIEnv *env, jstring string, jboolean *isCopy);
env: Java环境
string: jstring类型字符串
isCopy: 是否创建副本
这个方法与之配对的是ReleaseStringUTFChars()，这个方法会释放它，这个它是指通过这个方法转换后返回的char*，不是第二个参数
```
把鼠标放在v31上，然后按 N 键重命名

![](/Users/2h0n91i2hen/Pictures/douyin2/idapro_rename.png)

改成啥你自己觉得，我这里改成 charParams，还有a4也可以改了，接着文章后面出现一些变量可能被我改了的我只做简单提示。

接着看下面，这里调用了一个方法 ```sub_3E384(&v30);```，不知道干嘛的不要紧，我也不知道，双击点进去看看
```c++
_DWORD *__fastcall sub_3E384(_DWORD *result)
{
  *result = &algn_69AC4[8];
  return result;
}
```
algn_69AC4 这个东西没有在方法里声明，把鼠标放在上面一会可以看到 ```char[12]``` 是一个12位长度的字符数组，你可以把它当成字符串但不能叫它字符串；既然不知道那咋办嘛，我也不会，但我会调试，那就上调试看看呗，别把当前分析的程序关了；

## IDA调试

点 File -> New instance(新建实例) 新打开一个程序，大概长这样

![](/Users/2h0n91i2hen/Pictures/douyin2/idapro_new_instance.png)

然后点 GO，这个时候我们还需要做一件事，直接调试是不可能的，需要一个中介人一样的东西，和frida-server差不多的东西，一般都在ida安装文件夹里，我这里目前只有mac，所以只说mac，找到你的app

![](/Users/2h0n91i2hen/Pictures/douyin2/find_app.png)

然后右键显示包内容，路径是 Contents/MacOS/dbgsrc/ 

![](/Users/2h0n91i2hen/Pictures/douyin2/app_dbgsrc.png)

这里面有两个 android_server 一个是32位一个是64位，这里用32位就行，把它复制到你手机根目录的文件夹里，但也别乱放，我放在data里，命令如下：
```shell
adb push android_server /data/ # 复制
adb shell # 进入交互
$ su # 拿个权限
# chmod +x /data/android_server # 给个执行权限
# /data/android_server # 启动
IDA Android 32-bit remote debug server(ST) v1.22. Hex-Rays (c) 2004-2017
Listening on 0.0.0.0:23946...
=========================================================
[1] Accepting connection from 127.0.0.1...
[1] Closing connection from 127.0.0.1...
=========================================================
[2] Accepting connection from 127.0.0.1...
[2] Closing connection from 127.0.0.1...
```
输出了大概这么些东西就差不多了，其中23946是端口，复制一下；然后新开一个terminal，转发一下端口：
```shell
adb forward tcp:23946 tcp:23946
```
接着回到ida，在菜单栏处点 Debugger -> attach 选择 Remote ARMLinux/Android Debugger 调试类型

![](/Users/2h0n91i2hen/Pictures/douyin2/idapro_debugger_attach_remote_android_debugger.png)

在新弹出的窗口里点确定就行，如果你要改什么的话就点 options

![](/Users/2h0n91i2hen/Pictures/douyin2/idapro_debugger_setup.png)

不出意外的话你将会看到这个进程窗口，如果出意外的话我也没办法，大家都这么大了自己google

![](/Users/2h0n91i2hen/Pictures/douyin2/idapro_debugger_attach.png)

按 Ctrl-F mac也是这个快捷键，输入 com.ss 就会筛选处抖音进程，前提是你打开了抖音，出现这个窗口后你没打开，然后你打开了你就需要右键 Refresh 刷新再搜索，找到之后双击或者点 OK 开始debug，这里需要注意，如果你手机一直出现未响应弹窗记得要一直点等待，不然ida会崩，然后你就要重新来过

![](/Users/2h0n91i2hen/Pictures/douyin2/idapro_debugger_attach_find.png)

然后就进入了调试界面，这时候程序是卡着的，别乱点手机了

![](/Users/2h0n91i2hen/Pictures/douyin2/idapro_debugger_main.png)

第一步先加载 libuserinfo.so 找到基址，按 Ctrl-S 打开选择段跳转窗口

![](/Users/2h0n91i2hen/Pictures/douyin2/idapro_debugger_segment.png)

还是 ctrl-f 搜索，start 是开始，也就是起始地址，这就是基址

![](/Users/2h0n91i2hen/Pictures/douyin2/idapro_debugger_segment_find.png)

接着回到之前分析的ida，复制那个方法地址 0x3E384，用基址加这个得到方法在内存中的地址 0xcef08384，按 G 跳转到这地方

![](/Users/2h0n91i2hen/Pictures/douyin2/idapro_debugger_3E384_jump.png)

这些 DCB 是什么不清楚，但我知道按 P 就能显示正常汇编

![](/Users/2h0n91i2hen/Pictures/douyin2/idapro_debugger_dcb_to_asm.png)

在这方法开头处按 F2 打个断点，然后 F9 运行  

![](/Users/2h0n91i2hen/Pictures/douyin2/idapro_debugger_3E384_breakpoint.png)

这时候可以刷新一下让程序执行到这 ... emm 接着我调试了一会儿后发现这玩意儿好像没什么用，权当讲了下怎么调试吧，接着分析

----


```sub_3FCC8(&a1, charParams, (int)&v47);```第一个参数相当于返回值，char*类型；第二参数是url参数，第三个参数是一个char*，在这里这个方法返回的就是charparams，貌似什么都没做，进去看看
```c++
_DWORD *__fastcall sub_3FCC8(_DWORD *a1, const char *a2, int a3)
{
  char *v6; // r1

  v6 = (char *)-1;
  if ( a2 )
    v6 = (char *)&a2[strlen(a2)];
  *a1 = sub_3FA04(a2, v6, a3);
  return a1;
}
```
它这里有个骚操作，`v6 = (char *)&a2[strlen(a2)];`这行代码你怎么看都看不出是干嘛的，因为看得懂的不会看着文章，我一开始也是不知道干嘛的，看汇编是这样写的：
```
MOV     R0, R4
BLX     strlen      ; 获取R0字符串长度，保存在R0
ADDS    R1, R4, R0  ; R1 = R4 + R0；这里就是待处理字符串+字符串长度
```
通过strlen获取的是寄存器R0的长度，这返回的是一个整数，然后和R4也就是R0相加，看参数就该知道这个R4是个字符串，一个字符串和整数相加我**傻了；
但，不是这样，因为这里的ADDS不是把内容相加，为了探究搞懂是怎么回事我调试了一遍：
> 字符串存在R4寄存器，内容是manifest_version_code，长度肉眼可见21个字符，十六进制可写为0x15
> 首先经过第一条MOV指令后 R0,R1,R4 寄存器的地址分别是：`0xE12E6CE0、0xE12E6CE0、0xFFFFFFFF`
> 接着执行第二条跳转指令调用strlen方法后R0寄存器变成了0x00000015，这个如果当地址跳转是不存在的
> 最后执行第三条 ADDS 指令将R4和R0寄存器地址相加，`0xE12E6CE0 + 0x00000015 = 0xE12E6CF5`，所以R1就等于 0xE12E6CF5，这个地址的内容没意义

然后调用了 `sub_3FA04` 方法
> 第一个参数是字符串本身，
> 第二个参数是一个地址
> 第三个参数是一个整数

接着看 `sub_3FA04` 方法：
```c++
char *__fastcall sub_3FA04(_BYTE *a1, _BYTE *a2, int a3)
{
  _BYTE *v4; // r6
  int *v5; // r0
  int *v6; // r5
  void *v7; // r3
  char *result; // r0
  int *v9; // r0

  if ( a1 == a2 ) 
    return &algn_69AC4[8];
  if ( a1 )
  {
    v4 = (_BYTE *)(a2 - a1);      // a2地址-a1地址得到参数2长度，因为上一个方法对其相加过
    v5 = sub_3E9F4(a2 - a1, 0);
    v6 = v5;
    v7 = v5 + 3;
    if ( v4 == (_BYTE *)&dword_0 + 1 )
    {
      *((_BYTE *)v5 + 12) = *a1;
      goto LABEL_5;
    }
  }
  else
  {
    v9 = sub_3E9F4(0, 0);
    v4 = a1;
    v7 = v9 + 3;
    v6 = v9;
  }
  v7 = memcpy(v7, a1, (size_t)v4);
LABEL_5:
  result = (char *)v7;
  if ( v6 != &dword_69AC0 )
  {
    *v6 = (int)v4;
    v6[2] = 0;
    v4[(_DWORD)v7] = 0;
  }
  return result;
}
```
这里的参数 a3 是没意义的，根据ida的grahp view来分块分析，首先是入口：
```
.text:0003FA04 ; char *__fastcall sub_3FA04(_BYTE *a1_url_param_key, _BYTE *a2, int a3)
.text:0003FA04 sub_3FA04
.text:0003FA04 ; __unwind {
.text:0003FA04 CMP             R0, R1  ; Set cond. codes on Op1 - Op2
.text:0003FA06 MOV             R3, R1  ; Rd = Op2
.text:0003FA08 PUSH            {R4-R6,LR} ; Push registers
.text:0003FA0A ; 10:   v3 = a1;
.text:0003FA0A MOV             R4, R0  ; Rd = Op2
.text:0003FA0C ; 11:   if ( a1 == a2 )
.text:0003FA0C BEQ             loc_3FA56 ; B
```

为了方便查看分析，我会为每条分析都加上序号，以及我会打开ida自动注释
> 1、首先是对参数1和参数2的地址比较，看这两个参数是不是指向同一个地址，如果是它会跳转到这，然后程序结束。。。这肯定不是我们想要的
> > ```
.text:0003FA56 ; 12:     return &algn_69AC4[8];
.text:0003FA56
.text:0003FA56 loc_3FA56
.text:0003FA56 LDR             R3, =(dword_69AC0 - 0x3FA5C) ; Load from Memory
.text:0003FA58 ADD             R3, PC  ; dword_69AC0
.text:0003FA5A ADDS            R3, #0xC ; Rd = Op1 + Op2
.text:0003FA5C MOV             R0, R3  ; Rd = Op2
.text:0003FA5E POP             {R4-R6,PC} ; Pop reg
> > ```
> 2、接着看如果不等于会怎么跳转，其他的没什么好看的：
> > ```
.text:0003FA0E ; 13:   if ( a1 )
.text:0003FA0E CBZ             R0, loc_3FA60 ; Compare and Branch o
> >```
> 3、CBZ 指令是一个比较指令，如果比较的寄存器为零则跳转，这里的R0是参数1，也就是判断参数1是不是为0，或者说判断这个参数有没有内容或者是不是为null，先默认不为0，程序会执行到0x3fa10这：
> > ```
.text:0003FA10 ; 15:     v4 = a2 - a1;
.text:0003FA10 SUBS            R6, R1, R0 ; Rd = Op1 - Op2
.text:0003FA12 ; 16:     v5 = sub_3E9F4(a2 - a1, 0, a3, a2);
.text:0003FA12 MOVS            R1, #0  ; Rd = Op2
.text:0003FA14 MOV             R0, R6  ; Rd = Op2
.text:0003FA16 BL              sub_3E9F4 ; Branch with Link
.text:0003FA1A ; 17:     v6 = (int *)v5;
.text:0003FA1A CMP             R6, #1  ; Set cond. codes on Op1 - Op2
.text:0003FA1C MOV             R5, R0  ; Rd = Op2
.text:0003FA1E ; 18:     v7 = (void *)(v5 + 12);
.text:0003FA1E ADD.W           R3, R0, #0xC ; Rd = Op1 + Op2
.text:0003FA22 ; 19:     if ( v4 == 1 )
.text:0003FA22 BNE             loc_3FA48 ; Br
> > ```
> 4.1、首先是带进位的减法`SUBS`指令，R6 = R1 - R0，R0是参数1，R1是参数2，这里就可以得到之前用strlen得到的字符串长度，因为之前是用字符串本身地址 + 它长度的地址，而这个这个相加得到的新地址就是参数2，参数1是字符串本身，所以这里用参数2 - 字符串本身就能逆向得到字符串长度
> 4.2、然后两个MOV，后面是一个跳转，这个跳转的代码如下，首先说一下参数1 R0是字符串长度，参数2 R1是0从上面一眼就能看到：
> > ```
.text:0003E9F4 sub_3E9F4
.text:0003E9F4 ; __unwind {
.text:0003E9F4 MOV             R3, #0x3FFFFFFC
.text:0003E9FC CMP             R0, R3  ; R0 - R3
.text:0003E9FE PUSH            {R4,LR} ; 将R4, LR(R14)压入栈
.text:0003EA00 MOV             R4, R0  ; R4=R0
.text:0003EA02 BHI             loc_3EA46 ; 如果R0 > R3跳转
.text:0003EA04 CMP             R0, R1  ; R0 - R1
.text:0003EA06 BLS             loc_3EA36 ; 如果R0 <= R1跳转
.text:0003EA08 LSLS            R2, R1, #1 ; R1 << 1，差不多就是R1*2，结果放在R2
.text:0003EA0A CMP             R4, R2  ; R4 - R2，判断R4 > R2
.text:0003EA0C IT CC                   ; 如果R4 < R2，就执行MOVCC，否则跳过MOVCC
.text:0003EA0E MOVCC           R4, R2  ; 这里是将R1*2的值R2给R4
.text:0003EA10 ADD.W           R2, R4, #0x1D ; R2 = R4 + 0x1D
.text:0003EA14 CMP.W           R2, #0x1000 ; R2 - 0x1000；判断R2 > 0x1000
.text:0003EA18 ITE LS                  ; 如果R2 < 0x1000执行MOVLS
.text:0003EA1A MOVLS           R0, #0  ; 如果R2 < 0x1000，R0 = 0
.text:0003EA1C MOVHI           R0, #1  ; 如果R2 > 0x1000, R0 = 1
.text:0003EA1E CMP             R1, R4  ; R1 - R4,判断 R1 > R4
.text:0003EA20 IT CS                   ; 如果R1 < R4 执行 MOVCS
.text:0003EA22 MOVCS           R0, #0  ; R0 = 0
.text:0003EA24 CBZ             R0, loc_3EA36 ; 如果R0为0，也就是如果R1 < R4 跳转
.text:0003EA26 ADD.W           R4, R4, #0x1000 ; R4 += 0x1000
.text:0003EA2A UBFX.W          R2, R2, #0, #0xC ; R2 = (R2 & 0xFFF) >> 0
.text:0003EA2E SUBS            R4, R4, R2 ; R4 = R4 - R2
.text:0003EA30 CMP             R4, R3  ; R4 - R3; 判断R4 < R3，R3=0x3FFFFFFC
.text:0003EA32 IT CS                   ; 如果R4 > R3就执行MOVCS
.text:0003EA34 MOVCS           R4, R3  ; R4 > R3就把R3给R4
; 下面是两个跳转的代码
; loc_3EA36 是程序末尾返回部分，所以放在那个log输出前面以便于更好的查看，这个可以接着上面
.text:0003EA36 loc_3EA36               ; unsigned int
.text:0003EA36 ADD.W           R0, R4, #0xD  ; R0 = R4 + 0xD
.text:0003EA3A BL              _Znwj   ; operator new(uint) ; 分配内存到R0
.text:0003EA3E MOVS            R2, #0  ; Rd = Op2
.text:0003EA40 STR             R4, [R0,#4] ; 将R4寄存器中的数据写入 R0+0x4 这个地址
.text:0003EA42 STR             R2, [R0,#8] ; 将R2寄存器中的数据写入 R0+0x8 这个地址
.text:0003EA44 POP             {R4,PC} ; 将堆栈中的数据弹出到R4和PC中
;
; 这loc_3EA46就是一个log输出
.text:0003EA46
.text:0003EA46 loc_3EA46
.text:0003EA46 LDR             R0, =(aBasicStringSCr - 0x3EA4C) ; Load from Memory
.text:0003EA48 ADD             R0, PC  ; "basic_string::_S_create"
.text:0003EA4A BL              sub_3DBBC ; Branch with Link
.text:0003EA4A ; End of function sub_3E9F4
.text:0003EA4A
> > ```
> > 4.2.1、上面的注释是较早以前分析的时候写的，这个方法首先判断字符串长度是不是大于1073741820应该是判断有没有超过字符串最大长度；
> > 4.2.2、接着是判断字符串长度是不是大于0，这里是肯定，然后又判断字符串长度是不是小于参数2*2，这里的参数2是0，所以这是false
> > 4.2.3、然后是判断字符串长度 + 0x1D 后是不是大于4096


----

> 跳转到的地址汇编：
> > ```
.text:0003FA60 ; 27:     if ( a2 )
.text:0003FA60
.text:0003FA60 loc_3FA60
.text:0003FA60 CMP             R3, #0  ; Set cond. codes on Op1 - Op2
.text:0003FA62 BEQ             loc_3FA3C ; B
> > ```
> 4、这里有个CMP比较指令，这里是判断地址，这个R3是参数2，如果你回头去看传看可以知道这个参数是strlen结果存放的寄存器，如果R3 - 0等于0就跳转到0x3fa3c，不等于0就会执行一个log输出



