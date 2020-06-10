# 抖音分析篇(二) sub_3F044 方法逻辑

首先上伪代码
```c++
int __fastcall sub_3F044(int a1, const char *a2, int a3, int a4)
{
  size_t v6; // r0

  v6 = strlen(a2);
  return sub_3EF30((_BYTE **)a1, a2, v6);
}
```
就两个操作，获取参数2的长度，然后调用sub_3EF30，第一个参数时a1，第二个参数是字符串，第三个参数是字符串长度，跟进看到代码我就不想分析了，先来hook试试；注意第一个参数是个二级指针
```c++
int __fastcall sub_3EF30(_BYTE **a1, _BYTE *a2, size_t a3)
{
  _BYTE **v3; // r5
  _BYTE *v4; // r4
  size_t v5; // r6
  int v6; // r7
  int result; // r0
  int *v8; // r3

  v3 = a1;
  v4 = *a1;
  v5 = a3;
  v6 = *((_DWORD *)*a1 - 3);
  if ( a3 > 0x3FFFFFFC )
    android_log("basic_string::assign");
  if ( a2 < v4 || a2 > &v4[v6] || *((_DWORD *)v4 - 1) > 0 )
    return sub_3EF00(a1, 0, v6, a2, a3);
  if ( a3 > a2 - v4 )
  {
    if ( a2 == v4 )
      goto LABEL_9;
    if ( a3 != 1 )
    {
      memmove(v4, a2, a3);
      v4 = *v3;
      goto LABEL_9;
    }
LABEL_18:
    *v4 = *a2;
    v4 = *a1;
    goto LABEL_9;
  }
  if ( a3 == 1 )
    goto LABEL_18;
  memcpy(v4, a2, a3);
  v4 = *v3;
LABEL_9:
  v8 = &dword_69AC0;
  if ( v4 - 12 == (_BYTE *)&dword_69AC0 )
  {
    result = (int)v3;
  }
  else
  {
    *((_DWORD *)v4 - 3) = v5;
    LOBYTE(v8) = 0;
    result = (int)v3;
    *((_DWORD *)v4 - 1) = 0;
  }
  if ( v4 - 12 != (_BYTE *)&dword_69AC0 )
    v4[v5] = (_BYTE)v8;
  return result;
}
```
Hook 代码如下
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

var JNI_OnLoad;

var exports = Module.enumerateExportsSync("libuserinfo.so");
for (var i = 0; i < exports.length; i++) {
    var name = exports[i].name;
    var addr = exports[i].address;
    if (name == 'JNI_OnLoad') {
        JNI_OnLoad = addr;
    }
}

// Memory.readUtf8String
var mru8s = function(addr) {return Memory.readUtf8String(addr)}
// Memory.readPointer
var mrp = function(addr) {return Memory.readPointer(addr)}
// Memory.allocUtf8String
var mau8s = function(addr) {return Memory.allocUtf8String(addr)}

var BASE_ADDR = parseInt(JNI_OnLoad) - parseInt("0x14504");
var addr = '0x' + parseInt(BASE_ADDR + parseInt('0x3f044')).toString(16);

Interceptor.attach(new NativePointer(addr), {
    onEnter: function(args) {
        
    },
    onLeave: function(retval) {

    }
});

var i = 0;

var addr = '0x' + parseInt(BASE_ADDR + parseInt('0x3EF30')).toString(16);
Interceptor.attach(new NativePointer(addr), {
    onEnter: function(args) {
        // if (i < 2) {
            i ++;
            var msg = '===============【' + i + '】==================\n' + 
                        '[参数1 > {0}({3}/{4})\n[参数2 > {1}\n[参数3 > {2}\n'
                        .format(mru8s(mrp(args[0])), mru8s(args[1]), args[2], args[0], mrp(args[0]));
            send(msg);
        // }
    },
    onLeave: function(retval) {
        send('retval > {0}'.format(retval));
    }
});

Java.perform(function() {
    var UserInfo = Java.use("com.ss.android.common.applog.UserInfo");

    UserInfo.getUserInfo.implementation = function(arg0, arg1, arg2) {
        console.log('===================================');
        console.log("arg0 > ", arg0);
        console.log("arg1 > ", arg1);
        console.log("arg2 > ", arg2);
        var retval = this.getUserInfo(arg0, arg1, arg2);
        console.log("retval > ", retval);
        console.log('\n');
        return retval;
    }
});
```
python 脚本
```python
import frida
import sys


def on_message(message, data):
    if message['type'] == 'send':
        print("[*] {0}".format(message['payload']))
    else:
        print(message)


if __name__ == '__main__':
    jscode = open('3f044_hook.js', 'r').read()
    process = frida.get_usb_device().attach('com.ss.android.ugc.aweme')
    script = process.create_script(jscode)
    script.on('message', on_message)
    print('[*] Running CTF')
    script.load()
    sys.stdin.read()
```
部分输出
```shell
===================================
arg0 >  1573459557
arg1 >  https://api.amemv.com/aweme/v1/aweme/stats/?manifest_version_code=166&_rticket=1573459557529&ac=wifi&device_id=69143529399&iid=88263783316&os_version=9&channel=wandoujia&version_code=166&device_type=MI+8+UD&language=zh&uuid=868695035559432&resolution=1080*2029&openudid=ffa82bd3b1106594&update_version_code=1662&app_name=aweme&version_name=1.6.6&os_api=28&device_brand=Xiaomi&ssmix=a&device_platform=android&dpi=440&aid=1128&ts=1573459557
arg2 >  item_id,6757943951740718339,tab_type,0,play_delta,1,aweme_type,0,retry_type,no_retry,manifest_version_code,166,_rticket,1573459557529,ac,wifi,device_id,69143529399,iid,88263783316,os_version,9,channel,wandoujia,version_code,166,device_type,MI 8 UD,language,zh,uuid,868695035559432,resolution,1080*2029,openudid,ffa82bd3b1106594,update_version_code,1662,app_name,aweme,version_name,1.6.6,os_api,28,device_brand,Xiaomi,ssmix,a,device_platform,android,dpi,440,aid,1128
[*] ===============【89】==================
[参数1 > (0xcc36d02c/0xdbbe9acc)
[参数2 > efc84c17
[参数3 > 0x8

[*] retval > 0xcc36d02c
[*] ===============【90】==================
[参数1 > (0xcc36d08c/0xdbbe9acc)
[参数2 > 6757943951740718339
[参数3 > 0x13

[*] retval > 0xcc36d08c
[*] ===============【91】==================
[参数1 > (0xcc36d0bc/0xdbbe9acc)
[参数2 > 0
[参数3 > 0x1

[*] retval > 0xcc36d0bc
[*] ===============【92】==================
[参数1 > (0xcc36d0ec/0xdbbe9acc)
[参数2 > 1
[参数3 > 0x1

[*] retval > 0xcc36d0ec
[*] ===============【93】==================
[参数1 > (0xcc36d11c/0xdbbe9acc)
[参数2 > 0
[参数3 > 0x1

[*] retval > 0xcc36d11c
[*] ===============【94】==================
[参数1 > (0xcc36d14c/0xdbbe9acc)
[参数2 > no_retry
[参数3 > 0x8

[*] retval > 0xcc36d14c
[*] ===============【95】==================
[参数1 > 166(0xcc507fe4/0xbf1fdcec)
[参数2 > 166
[参数3 > 0x3

[*] retval > 0xcc507fe4
[*] ===============【96】==================
[参数1 > 1573459557529(0xcc312014/0xcf4fcd6c)
[参数2 > 1573459557529
[参数3 > 0xd

[*] retval > 0xcc312014
[*] ===============【97】==================
[参数1 > wifi(0xcc31202c/0xcc31203c)
[参数2 > wifi
[参数3 > 0x4

[*] retval > 0xcc31202c
[*] ===============【98】==================
[参数1 > 69143529399(0xcc312074/0xcc312054)
[参数2 > 69143529399
[参数3 > 0xb

[*] retval > 0xcc312074
[*] ===============【99】==================
[参数1 > 88263783316(0xcc3120bc/0xcc31209c)
[参数2 > 88263783316
[参数3 > 0xb

[*] retval > 0xcc3120bc
[*] ===============【100】==================
[参数1 > 9(0xcc3137cc/0xbf1fde2c)
[参数2 > 9
[参数3 > 0x1

[*] retval > 0xcc3137cc
[*] ===============【101】==================
[参数1 > wandoujia(0xcc3138ec/0xcc3137f4)
[参数2 > wandoujia
[参数3 > 0x9

[*] retval > 0xcc3138ec
[*] ===============【102】==================
[参数1 > 166(0xcc313d3c/0xbf1fde3c)
[参数2 > 166
[参数3 > 0x3

[*] retval > 0xcc313d3c
[*] ===============【103】==================
[参数1 > MI+8+UD(0xcc3142c4/0xcc31425c)
[参数2 > MI 8 UD
[参数3 > 0x7

[*] retval > 0xcc3142c4
[*] ===============【104】==================
[参数1 > zh(0xcc3142f4/0xbf1fde4c)
[参数2 > zh
[参数3 > 0x2

[*] retval > 0xcc3142f4
[*] ===============【105】==================
[参数1 > 868695035559432(0xcc31433c/0xcf4fccec)
[参数2 > 868695035559432
[参数3 > 0xf

[*] retval > 0xcc31433c
[*] ===============【106】==================
[参数1 > 1080*2029(0xcc314384/0xcc314364)
[参数2 > 1080*2029
[参数3 > 0x9

[*] retval > 0xcc314384
[*] ===============【107】==================
[参数1 > ffa82bd3b1106594(0xcc3143b4/0xdbc3a92c)
[参数2 > ffa82bd3b1106594
[参数3 > 0x10

[*] retval > 0xcc3143b4
[*] ===============【108】==================
[参数1 > 1662(0xcc3143e4/0xcc3143c4)
[参数2 > 1662
[参数3 > 0x4

[*] retval > 0xcc3143e4
[*] ===============【109】==================
[参数1 > aweme(0xcc314474/0xcc314454)
[参数2 > aweme
[参数3 > 0x5

[*] retval > 0xcc314474
[*] ===============【110】==================
[参数1 > 1.6.6(0xcc3144bc/0xcc31449c)
[参数2 > 1.6.6
[参数3 > 0x5

[*] retval > 0xcc3144bc
[*] ===============【111】==================
[参数1 > 28(0xcc3144d4/0xbf1fde5c)
[参数2 > 28
[参数3 > 0x2

[*] retval > 0xcc3144d4
[*] ===============【112】==================
[参数1 > Xiaomi(0xcc314504/0xcc507f64)
[参数2 > Xiaomi
[参数3 > 0x6

[*] retval > 0xcc314504
[*] ===============【113】==================
[参数1 > a(0xcc31451c/0xbf1fde6c)
[参数2 > a
[参数3 > 0x1

[*] retval > 0xcc31451c
[*] ===============【114】==================
[参数1 > android(0xcc31454c/0xcc507f7c)
[参数2 > android
[参数3 > 0x7

[*] retval > 0xcc31454c
[*] ===============【115】==================
[参数1 > 440(0xcc3145f4/0xbf1fde8c)
[参数2 > 440
[参数3 > 0x3

[*] retval > 0xcc3145f4
[*] ===============【116】==================
[参数1 > 1128(0xcc507f9c/0xcc314b8c)
[参数2 > 1128
[参数3 > 0x4

[*] retval > 0xcc507f9c
retval >  a13541aca5964d26291468dd515993c069e1
```
首先可以看出在伪代码中这个方法的第一个参数就是返回值，因为地址是一样的；然后第二个参数就是请求参数的value；第三个参数是这个value的长度，十六进制格式的；应该看得懂吧...

然后不知道有没有发现这个方法第一次调用时的参数有点特殊
```
[*] ===============【89】==================
[参数1 > (0xcc36d02c/0xdbbe9acc)
[参数2 > efc84c17
[参数3 > 0x8
```
这个参数是在这个so里添加的，请求的链接中是没有的，不信可以看。然后这个value所对应的key叫`rstr`
