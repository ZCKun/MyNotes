# 抖音分析篇(二) getName

## 说在前面的话
> 本来想只看汇编来一行行的分析的，其实我还学了近半个月的汇编。。。不过看到这个算法的汇编代码我第一个想法就是放弃，下次吧，(:
> 说实话这个getName很简单，本来想从汇编开始分析的。。。但我看到那个排序算法我就头大，这次就放弃吧

----
## 正文

先来伪代码

```c++
char *__fastcall getName(int v25_serverTime, char *params, char *a3)
{
  char *v3; // r4
  char *v4; // ST14_4
  char v9; // [sp+18h] [bp-10h]

  std::unique_lock<std::mutex>::unique_lock(&v9, &mtx);
  if ( (unsigned __int8)inited ^ 1 )
  {
    as[0] = 0;
    std::unique_lock<std::mutex>::unlock(&v9);
    v3 = as;
  }
  else
  {
    if ( v25_serverTime < 0 )
      v25_serverTime = time(0);
    if ( params && a3 )
    {
      memset(&concat_buffer, 0, 4097u);         // 将concat_buffer地址后面4097个字节全部用0填充。。刚好填满
      snprintf((char *)&concat_buffer, 4096u, "%s", params);// 格式化字符串；将params全部放入concat_buffer，因为concat_buffer长度远大于params，所以后面会自动加结束符'\0'
      v4 = getName(v25_serverTime, (char *)&concat_buffer);
      std::unique_lock<std::mutex>::unlock(&v9);
      v3 = v4;
    }
    else
    {
      as[0] = 0;
      std::unique_lock<std::mutex>::unlock(&v9);
      v3 = as;
    }
  }
  std::unique_lock<std::mutex>::~unique_lock(&v9);
  return v3;
```
> 第一个参数时间戳
> 第二参数是拼接后的url参数中的value，不过这里的这个value有点不同，它拼接的时候`排序`过，`顺序就是按照参数key的顺序(a-z)排序的`，用`python sorted`就知道了
> 第三个参数是个空参数
> 返回结果就是我们要的东西了
> 参数结果hook代码如下
```javascript
String.prototype.format = function () {
    var values = arguments;
    return this.replace(/\{(\d+)\}/g, function (match, index) {
        if (values.length > index) {
            return values[index];
        } else {
            return "";
        }
    });
}
// Memory.readUtf8String
var mru8s = function(addr) {return Memory.readUtf8String(addr)}
// Memory.readPointer
var mrp = function(addr) {return Memory.readPointer(addr)}
// Memory.allocUtf8String
var mau8s = function(addr) {return Memory.allocUtf8String(addr)}


var JNI_OnLoad;
var exports = Module.enumerateExportsSync("libuserinfo.so");
for (var i = 0; i < exports.length; i++) {
    var name = exports[i].name;
    var addr = exports[i].address;
    if (name == 'JNI_OnLoad') {
        JNI_OnLoad = addr;
    }
}

var BASE_ADDR = parseInt(JNI_OnLoad) - parseInt("0x14504");
var addr = '0x' + parseInt(BASE_ADDR + parseInt('0x11828')).toString(16);

var i = 0;
Interceptor.attach(new NativePointer(addr), {
    onEnter: function(args) {
        i++;
        var msg = '===============' + i + '===============\n' +
                    '参数1 > {0}\n参数2 > {1}\n参数3 > {2}\n'
                    .format(args[0], mru8s(args[1]), mru8s(args[2]));
        console.log(msg);
    },
    onLeave: function(retval) {
        console.log('retval > ', mru8s(retval));
    }
});
```
输出
```
[*] Running CTF
===============1===============
参数1 > 0x5dc91e39
参数2 > 1573461562225wifi1128awemewandoujia200Xiaomi69143529399androidMIa8aUD44088263783316zh166ffa82bd3b11065942891080*2029no_retryefc84c17a157346156116628686950355594321661.6.6
参数3 > 

retval >  a13541cc39834d8e691b33d3509c9dcae1e1
```
从这个getName伪代码中可以看到又调用了一个名为getName的方法，其伪代码以及不知道我什么是时候写的注释如下
```c++
// 这个方法应该是个加密方法
char *__fastcall getName(int serverTime, char *params)
{
  signed int i; // [sp+Ch] [bp-1Ch]
  signed int j; // [sp+10h] [bp-18h]
  signed int k; // [sp+14h] [bp-14h]
  signed int l; // [sp+18h] [bp-10h]

  if ( (unsigned __int8)inited ^ 1 )
  {
    as[0] = 0;
  }
  else if ( params )
  {
    sprintf(shuffle1, "%08x", serverTime);      // 格式化字符串，将时间戳转为16进制
    sprintf(shuffle2, "%08x", serverTime);      // 同上
    shuffle((int)shuffle1, (char *)&array1);    // 这个是将shuffle1随机排序，array1=57218436
    shuffle((int)shuffle2, (char *)&array2);    // 同上，array2=15387264
    getMd5(params);                             // 这里将url参数进行md5加密，加密的结果是md5_value
    if ( serverTime & 1 )                       // 这里是判断时间戳是不是2的倍数
      getMd5(md5_value);                        // 是的话就再加密一次
    as[0] = 97;                                 // 先将as数组第一位添加字符a
    byte_612C5 = version;                       // 这里实际是as[1] = version；version=1
    for ( i = 0; i <= 7; ++i )                  // 接着循环7次，将md5加密后的params的前7个字符添加到as字符数组，as中存放不是依次存放，以2*(i+1)索引存放
      as[2 * (i + 1)] = md5_value[i];
    for ( j = 0; j <= 7; ++j )                  // 将第二次随机排序后的时间戳前7位取出，以(2*j+3)索引存放到as字符数组中
      as[2 * j + 3] = shuffle2[j];
    byte_612D6 = 0;
    for ( k = 0; k <= 7; ++k )                  // 将第一次随机排序后的时间戳前7位取出，以(2*k)索引存放到cp字符数组中
      cp[2 * k] = shuffle1[k];
    for ( l = 0; l <= 7; ++l )                  // 将md5加密后的params按索引l+24取出，以(2*l+1)索引存放到cp字符数组
      cp[2 * l + 1] = md5_value[l + 24];
    byte_6139C = 101;                           // cp[16] = 0x65；0x65十进制101，在ascii中是字符e
    byte_6139D = version;                       // cp[17] = version；version=1
    byte_6139E = 0;                             // cp[18] = 0
    strncpy(&byte_612D6, cp, 0x12u);            // 将字符数组cp的前18位也就是复制到byte_612D6所在的地址
  }
  else
  {
    as[0] = 0;
  }
  return as;
}
```
这里我们只需要看那个排序方法shuffle
```c++
// 字符串随机排序
int __fastcall shuffle(int result, char *a2)
{
  byte_60123 = *(_BYTE *)(result + 7);
  byte_6011E = *(_BYTE *)(result + 2);
  byte_60120 = *(_BYTE *)(result + 4);
  byte_60122 = *(_BYTE *)(result + 6);
  byte_6011F = *(_BYTE *)(result + 3);
  byte_60121 = *(_BYTE *)(result + 5);
  byte_6011D = *(_BYTE *)(result + 1);
  tmp[0] = *(_BYTE *)result;
  *(_BYTE *)result = tmp[(unsigned __int8)*a2 - 49];
  *(_BYTE *)(result + 2) = tmp[(unsigned __int8)a2[2] - 49];
  *(_BYTE *)(result + 4) = tmp[(unsigned __int8)a2[4] - 49];
  *(_BYTE *)(result + 6) = tmp[(unsigned __int8)a2[6] - 49];
  *(_BYTE *)(result + 5) = tmp[(unsigned __int8)a2[5] - 49];
  *(_BYTE *)(result + 3) = tmp[(unsigned __int8)a2[3] - 49];
  *(_BYTE *)(result + 1) = tmp[(unsigned __int8)a2[1] - 49];
  *(_BYTE *)(result + 7) = tmp[(unsigned __int8)a2[7] - 49];
  *(_BYTE *)(result + 8) = 0;                   // 这里不是数字0，这是结束符号\0
  return result;
}
```
排序方法对应python代码
```python
def shuffle(result, array):
    ret = ['0', '0', '0', '0', '0', '0', '0', '0']
    ret[2] = result[ord(array[2]) - 49]
    ret[4] = result[ord(array[4]) - 49]
    ret[6] = result[ord(array[6]) - 49]
    ret[5] = result[ord(array[5]) - 49]
    ret[3] = result[ord(array[3]) - 49]
    ret[1] = result[ord(array[1]) - 49]
    ret[7] = result[ord(array[7]) - 49]
    ret[0] = result[ord(array[0]) - 49]
    return ''.join(ret)
```
getName对应python代码
```python
def get_name(server_time, params):
    _as = ['0', '0', '0', '0', '0', '0', '0', '0', '0', '0', '0', '0', '0', '0', '0', '0', '0', '0']
    _cp = ['0', '0', '0', '0', '0', '0', '0', '0', '0', '0', '0', '0', '0', '0', '0', '0', '0', '0']
    shuffle1 = hex(server_time)[2:]
    shuffle2 = hex(server_time)[2:]
    shuffle1 = shuffle(shuffle1, '57218436')
    shuffle2 = shuffle(shuffle2, '15387264')
    params_md5 = hashlib.md5(params.encode('utf-8')).hexdigest()
    if server_time & 1:
        params_md5 = hashlib.md5(params_md5.encode('utf-8')).hexdigest()
    _as[0] = chr(97)
    _as[1] = chr(49)
    for i in range(8):
        _as[2*(i+1)] = params_md5[i]

    for j in range(8):
        _as[2*j+3] = shuffle2[j]

    for k in range(8):
        _cp[2*k] = shuffle1[k]

    for l in range(8):
        _cp[2*l+1] = params_md5[l+24]
    
    _cp[16] = chr(101)
    _cp[17] = chr(49)

    return ''.join(_as + _cp)
```
我已经测试了很多次了，和hook到的结果一毛一样，当然不信的话可以用上面hook到的结果测试

----

## 最后
emmm，这个最后获取结果好像就这些了吧（好敷衍呀），等我啥时候想起来要补充的再添加吧，太饿了。。。。
