# 抖音分析篇(二) Request Params


先分析个`if`块，对应的c++伪代码如下
```c++
  if ( paramsArray )                            // 如果String[]不为null
  {
    pArrayLength = _JNIEnv::GetArrayLength(env, paramsArray);// 获取String[]长度
    if ( !(pArrayLength & 1) )                  // 这里是判断String[]长度是不是2的倍数，目的应该是为了判断参数是否完整
    {
      for ( j = 0; j < pArrayLength; j += 2 )   // 以2为间隔遍历String[]，其实就是为了获取key，因为key肯定在前面的，例如：0 2 4 6 8 10 都是key的索引
      {
        key = _JNIEnv::GetObjectArrayElement((int)env, (int)paramsArray, j);// GetObjectArrayElement这个方法是获取array元素，第一个参数是jnienv，参数二是array，参数三是索引
        key_char = (const char *)_JNIEnv::GetStringUTFChars((int)env, key, 0);// 将array元素转为字符串
        value = _JNIEnv::GetObjectArrayElement((int)env, (int)paramsArray, j + 1);// 这个就是获取value了
        v38_value = getUTF8(env, (int)jcls, value);// 转为字符串
        nullsub_1();
        sub_3FCC8(&v46, key_char, (int)&v45);   // 首先处理key，v45为key_char长度
        v18 = std::map<std::string,std::string,std::less<std::string>,std::allocator<std::pair<std::string const,std::string>>>::operator[](
                (int)&v47,
                (int)&v46);                     // 将处理后的key添加到map
        sub_3F044(v18, v38_value, v19, v18);    // 接着处理value
        sub_3EAE8(&v46);
        nullsub_2();
        _JNIEnv::ReleaseStringUTFChars(env, key, key_char);
      }
    }
  }
```
注释因为是很久以前写的了，仅供参考。
map这玩意儿类似`Java中的map`和`python中的dict`，不过底层是啥玩意儿我就不太清楚，没怎么学过c++。
这里的逻辑应该看得明白：
> 1、首先是判断paramsArray，也就是请求参数被分割成的字符串数组。这个字符串数据的索引%2(0,2,...)是key，差不多长这样：`[key,value,key,value,...]`
> 2、然后就是判断参数数组是否完整
> 3、接着就是遍历字符串数组，按键值对的形式保存到map里

挑几条代码分析一波
> 这里是调用了jni的`GetObjectArrayElement`方法获取数组元素，第一个参数是`java接口指针`，第二参数是`jobjectArray数组`，第三个参数是`索引`。分别获取key和value
```c++
key = _JNIEnv::GetObjectArrayElement((int)env, (int)paramsArray, j);
value = _JNIEnv::GetObjectArrayElement((int)env, (int)paramsArray, j + 1);
```
> 然后呢，v47是map类型的。首先是operator[]方法，这个方法`返回到映射到等于 key 的关键的值的引用，若这种关键不存在则进行插入`。至于3F044这个地址的方法是干嘛的我就懒得取了解了，只要大概知道做了些什么就行。
```c++
v18 = std::map<std::string,std::string,std::less<std::string>,std::allocator<std::pair<std::string const,std::string>>>::operator[](
                (int)&v47,
                (int)&v46);
sub_3F044(v18, v38_value, v19, v18);
```
----
后面还有几行也一并分析了
```c++
v41 = &v47;
v45 = std::map<std::string,std::string,std::less<std::string>,std::allocator<std::pair<std::string const,std::string>>>::begin(&v47);// 获取map首个iterate
v46 = std::map<std::string,std::string,std::less<std::string>,std::allocator<std::pair<std::string const,std::string>>>::end(v41);// 获取map最后一个iterate
while ( std::_Rb_tree_iterator<std::pair<std::string const,std::string>>::operator!=(&v45, &v46) )// 如果首尾不相等就一直循环
{
v42 = std::_Rb_tree_iterator<std::pair<std::string const,std::string>>::operator*(&v45);
sub_3F36C((int)&v32, v42 + 4);              // 字符串拼接
std::_Rb_tree_iterator<std::pair<std::string const,std::string>>::operator++(&v45);
}
v20 = sub_3EC48(&v32);
v21 = sub_3EC80(&v32);
sub_153C0(v20, v21, v6);
v22 = (char *)sub_3E4D4((int)&v32);
```
也是挑选几行分析
> 1、这两行是获取map的开头和结尾两个迭代器
```c++
v45 = std::map<std::string,std::string,std::less<std::string>,std::allocator<std::pair<std::string const,std::string>>>::begin(&v47);
v46 = std::map<std::string,std::string,std::less<std::string>,std::allocator<std::pair<std::string const,std::string>>>::end(v41);
```
> 2、然后呢。先是一层while循环，条件是v45和v46不相等就一直循环
```c++
while ( std::_Rb_tree_iterator<std::pair<std::string const,std::string>>::operator!=(&v45, &v46) )// 如果不相等就一直循环
{
v42 = std::_Rb_tree_iterator<std::pair<std::string const,std::string>>::operator*(&v45);
sub_3F36C((int)&v32, v42 + 4);              // 字符串拼接
std::_Rb_tree_iterator<std::pair<std::string const,std::string>>::operator++(&v45);
}
```
> 循环内部首先是从map开头迭代器中获取key的name，然后用获取到的name的地址偏移4个字节获取到下一个key的name。以上仅仅是我个人的分析，所以为了验证，直接上hook


