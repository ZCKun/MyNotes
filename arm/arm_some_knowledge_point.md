# ARM部分知识点


**S**：指令执行后程序状态寄存器的条件标志位将被刷新 ， 如 ```ADDS R1,R0,#2```
**!**：指令中的地址表达式中含有!后缀时，指令执行后，基址寄存器中的地址值将发生变化，变化的结果是：**基址寄存器中的值(指令执行后)=指令执行前的值 + 地址偏移量**，如 ```LDR R3,[R0,#2]!```, 指令执行后 R0 地址 = R0 + 2

----
