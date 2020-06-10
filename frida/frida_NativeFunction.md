# Frida - NativeFunction

其实我写了两篇文章，但是第一篇中我觉得有些东西太过于负责，要去阅读源码，加上我害怕自己没讲好误导他人，当然这篇文章也是，如果有错误的地方请指出，我在确认后会更改的，谢谢。



为了搞懂 **NativeFunction** 的参数到底怎么写，我拿出了我许久未碰的 **AndroidStudio** 准备大干一场 —— 更新到最新版。

然后创建了一个名为 **NDKApplication** 的app，其包名为 **com.n112h0ng.ndkapplication**。

整个AS是这样的

![](/Users/2h0n91i2hen/Pictures/frida/nativefunction/ndkapplication_in_as_home.png)

正在更新一些东西，可以无视，待会就好了。

关于as下载很慢，比如这些host
```
jcenter.bintray.com
dl.google.com
```
解决办法是先去 http://ping.chinaz.com ping一下host，得到一个然后从中找一个延迟最低的ip(国内的，你找国外的就是人才了)，然后添加到host就行；

我把我改过的文件内容直接贴上来
首先是 **activity_main.xml** 布局文件
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <TextView
            android:id="@+id/sample_text"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Hello World!"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintLeft_toLeftOf="parent"
            app:layout_constraintRight_toRightOf="parent"
            app:layout_constraintTop_toTopOf="parent" />

        <!-- 就加了个按钮 -->
        <Button
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="click"
            android:id="@+id/btn" />

    </LinearLayout>

</androidx.constraintlayout.widget.ConstraintLayout>
```
**MainActivity.java**
```java
package com.n112h0ng.ndkapplication;

import androidx.appcompat.app.AppCompatActivity;


import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity {
    private TextView tv;
    private Button btn;
    
    private int number;

    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");  // 加载lib
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main); // 关联布局文件

        // 初始化
        tv = findViewById(R.id.sample_text);
        btn = findViewById(R.id.btn);
        number = 0;
        btn.setOnClickListener(new View.OnClickListener() {
            public void onClick(View v) {
                a();
            }
        });
    }

    private void a() {
        byte[] name = new byte[]{'2', 'h', '0', 'N', 'g'};
        tv.setText(stringFromJNI(name, "20", number));
        number++;
    }

    /**
     * A native method that is implemented by the 'native-lib' native library,
     * which is packaged with this application.
     * @param b byte数组类型的name
     * @param s String字符串类型的age
     * @param n int类型计数用的
     * @return 字符串
     */
    public native String stringFromJNI(byte[] b, String s, int n);
}
```
最后是 **native-lib.cpp**
```C++
#include <jni.h>
#include <string>

#define C_A_SIZE 1024

using namespace std;

extern "C" JNIEXPORT jstring JNICALL
Java_com_n112h0ng_ndkapplication_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject thiz,
        jbyteArray name,
        jstring age,
        jint number) {
    // 获取byte数组也就是name的长度
    int nameLen = env->GetArrayLength(name);
    // 然后根据长度将byte数组转为char*字符串
    char *cName = new char[nameLen];
    env->GetByteArrayRegion(name, 0, nameLen, reinterpret_cast<jbyte*>(cName));

    // 接着将jstring转为char*字符串
    const char *cAge = env->GetStringUTFChars(age, 0);

    // 这是c++里string。就是字符串
    string hello = "Hello from C++";

    // 将 number、hello、name、age 拼接成一个新字符数组 ret
    char ret [C_A_SIZE];
    sprintf(ret, "[%d]%s%s %s", number, hello.c_str(), cName, cAge);

    // 释放内容
    env->ReleaseStringUTFChars(age, cAge);
    // 通过NewStringUTF转为jstring后返回
    return env->NewStringUTF(ret);
}
```
我去录个gif，稍等...
效果如图，图片差不多4mb。。垃圾服务器可能加载很慢

![](/Users/2h0n91i2hen/Pictures/frida/nativefunction/ndkapplication_run_1.gif)


emmm，现在是10月13日，昨天和今天下午肝番去了，太鸡儿好看了给忘了...扯远了。






