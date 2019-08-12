## 前言

不记得是哪一天，忽忽悠悠地就进入了某猫社区（你懂的），从此，每天早上一瓶营养快线。庆幸的是，该社区为了盈利，开启了VIP通道和播放次数限制，不然可以直接喝蛋白质了。不过正值青春、精力旺盛的我们，怎么能让理智控制欲望？那就成为高大上的会员，开启VIP加速通道，无限观看！充钱？充钱是不可能充钱的，这辈子都不可能充钱。作为Android开发者，应该用特殊的手段来搞定特殊的事情，带着我们目标，那就来一次Android的逆向之旅吧！    

发布此文的目的，除了分享整个破解过程外，还希望可以帮助你打开Android逆向的大门，体验不一样的Android世界。
    
...没时间解释了，赶紧上车吧..

## 准备工作

玩逆向的基本工具，这里需要准备Android逆向三件套 **[apktool](https://ibotpeaches.github.io/Apktool/)**、**[dex2jar](https://sourceforge.net/projects/dex2jar/)**、**[jd-gui](http://java-decompiler.github.io/)**

apktool：反编译apk、重新打包新apk。你可以得到 **smali、res、AndroidManifest.xml** 等文件；

dex2jar：把Android执行的 **dex** 文件转成 **jar** 文件；

jd-gui：一款可以方便阅读 **jar** 文件的代码工具。

他们之间的关系，可以参考下图：

<img src="https://user-gold-cdn.xitu.io/2019/8/9/16c7599d4ee4648d?w=690&h=411&f=jpeg&s=39307" />

> 下载好上面三个工具，就可以开启你的逆向之旅了！

## 逆向开始

> 提到逆向，可能很多朋友会想到Xposed，用Xposed去Hook参数，绕过if判断。没错，用Xposed确实很容易就能达到目的，但是它的限制比较大，手机需要root，并且安装Xposed模块，或者需要跑在VirtualXposed虚拟环境下，增加了使用者的上手成本。

我们这里的破解方式，会直接输出一个破解版Apk，使用者不需要进行任何多余的操作，安装即可使用。

### 第一步：将原应用 apk 后缀改成 zip，解压出 classes.dex 文件

<img src="https://user-gold-cdn.xitu.io/2019/8/10/16c7b6669657559d?w=172&h=106&f=png&s=8524" />

</br>

<img src="https://user-gold-cdn.xitu.io/2019/8/9/16c75c62807636e9?w=660&h=367&f=png&s=21694" width="440" hegiht="244" align=center />
    
> 其实这一步是最难的，逆向最难的在于脱壳，脱壳分两种，手动脱壳(手脱)和机器脱壳(机脱)。什么是脱壳呢？就是很多App在发布到应用市场之前，会进行加固，即加壳，它会把真正的dex文件给"藏"起来，我们就需要通过脱壳的方式去找到应用里真正的dex，才能拿到里面的源码。只有拿到源码并读懂源码(混淆后连蒙带猜)，才能找到爆破点，才能修改代码重新编译。

>我们这里破解的某猫App，因为它的特殊性质，它上不了市场，也没有加固，最可气的是，它居然不混淆，这让我们破解的难度直线下降。
    
### 第二步：使用 dex2jar 将 classes.dex 转成 jar 文件
 
 cmd到dex2jar文件夹目录，执行
 
```cmd
    d2j-dex2jar D://xxx/xxx/classes.dex
```

得到 **jar** 文件

<img src="https://user-gold-cdn.xitu.io/2019/8/9/16c75d87edd7ef60?w=596&h=28&f=png&s=2572" width="596" hegiht="28" align=center />

### 第三步：将 jar 文件用 jd-gui 打开，查看源代码

<img src="https://user-gold-cdn.xitu.io/2019/8/9/16c75daa65e3f220?w=875&h=635&f=png&s=91460" width="437" hegiht="317" align=center />

## 静态分析

拿到源码后，首先我们需要找到应用的限制点，绕过App里面的判断。

<img src="https://user-gold-cdn.xitu.io/2019/8/10/16c7b7bb608c1837?w=540&h=960&f=jpeg&s=22805" width="270" hegiht="480" align=center />

然后分析源码，该从哪里开始入手呢？

我们都知道，一个完整Android应用，可能会存在各种第三方，各种依赖库，这些依赖都会被编译到dex里面，所以这个Jar包里面会存在很多不同包名的类文件，为了方便找到破解应用的包名，我们可以借助adb打印栈顶activity的类全路径：

```cmd
    adb shell dumpsys activity | findstr "mFocusedActivity"
```

<img src="https://user-gold-cdn.xitu.io/2019/8/10/16c7b8b4b0b58291?w=673&h=261&f=png&s=27446"/>

activity的包路径已经打印出来了，接下来在 jar 文件里面找到 **PlayLineActivity.java** 的相关代码。

根据页面Toast提示，很轻松就能定位到爆破点。

<img src="https://user-gold-cdn.xitu.io/2019/8/10/16c7b8e55ce67c75?w=837&h=516&f=png&s=81619"/>

```java
    UserUtils.getUserInfo().getIs_vip().equals("1")
```

可以看出，当会员字段为 1 时，说明是会员用户，就会切换至线路2。

```java
    Hawk.put("line", "2");
```

那接下来只需要修改用户实体类 **UserModel** 的 **getIs_vip()** 方法，让它永远返回 **1** 就行了。

## 破解

**dex2jar、jd-gui** 都只是分析工具，下面才是真正破解的开始。

### Smali简介

> Dalvik虚拟机和Jvm一样，也有自己的一套指令集，类似汇编语言，但是比汇编简单许多。我们编写的Java类，最后都会通过虚拟机转化成Android系统可以解读的smali指令，生成后缀为 .smali 的文件，与Java文件一一对应（也可能会比Java文件多，典型的比如实现某个接口的匿名内部类），这些smali文件就是Dalvik的寄存器语言。 只要你会java，了解android的相关知识，就能轻松的阅读它，

所以，我们真正需要修改的东西，是 java 代码对应的 smali 指令。

### 反编译

我们利用apktool工具，来提取apk里面的 smali文件。

cmd到apktool文件夹下面，执行 （你也可以配置环境变量，这样会方便一些）

```cmd
    apktool.bat d -f [apk输入路径] [文件夹输出路径]
```

<img src="https://user-gold-cdn.xitu.io/2019/8/10/16c7bacf87ce677b?w=697&h=110&f=png&s=16031"/>

反编译成功后，打开smali文件夹，找到 **UserModel.java** 对应包名下的 **UserModel.smali** 文件。

### 爆破

找到了爆破文件，找到了爆破点，接下来就可以对 **UserModel.smali** 文件进行爆破了（为什么叫爆破，我也不知道，行内都是这样叫的，感觉高大上，其实就是修改文件）。

用编辑器打开 **UserModel.smali** ，找到 **getIs_vip** 方法

<img src="https://user-gold-cdn.xitu.io/2019/8/10/16c7bb829aa1eb38?w=988&h=220&f=png&s=16335"/>

可以看到，它返回了成员变量 **is_vip** 的值，我们只需要把它的返回值修改成 **1** 就行了。

> 如果对smali指令不熟悉，你可以花10分钟去了解一下smali的基本语法。

<img src="https://user-gold-cdn.xitu.io/2019/8/10/16c7bba10c172083?w=973&h=273&f=png&s=17650"/>

定义一个string类型的常量 **v1**，赋值为 **1**，并将它返回出去。

### 动态调试

破解的这个好像太简单了，都省掉了调试步骤，那就直接

**保存，搞定！**

## 回编

接下来把反编译生产的文件夹又重新回编成 apk。

### 重新打包

cmd到apktool文件夹下面，执行

```cmd
    apktool b [文件夹输入路径] -o [apk输出路径]
```

如果修改smali文件没有问题的话，就可以正常生成一个新的 apk 文件。

<img src="https://user-gold-cdn.xitu.io/2019/8/10/16c7bc841441cecd?w=352&h=112&f=png&s=12086"/>

这时候直接将重新打包的apk文件拿去安装是不行的，因为之前zip解压的目录中，**META-INF** 文件夹就是存放签名信息，为了防止恶意串改。

**所以我们需要对重新打包的apk重新签名。**

### 重新签名

首先准备一个 **.jks** 的签名文件，这个开发android的同学应该很熟悉了。

配置了JDK环境变量，直接执行：

```cmd
    jarsigner -verbose -keystore [签名文件路径] -storepass [签名文件密码] -signedjar [新apk输出路径] -digestalg SHA1 -sigalg MD5withRSA [旧apk输入路径] [签名文件别名]
```

<img src="https://user-gold-cdn.xitu.io/2019/8/10/16c7bd533315f491?w=668&h=199&f=png&s=27954"/>

最后在你的文件夹下面，就可以看到一个 **某猫VIP破解版.apk**。

安装并验证功能

<img src="https://user-gold-cdn.xitu.io/2019/8/10/16c7bdf4d548b877?w=540&h=960&f=jpeg&s=21468" width="270" hegiht="480" align=center />
<img src="https://user-gold-cdn.xitu.io/2019/8/10/16c7bdc646e4b4b7?w=720&h=1280&f=jpeg&s=19961" width="270" hegiht="480" align=center />

## 总结

最后来梳理一下破解流程：

**1、将原应用 apk 后缀改成 zip，解压出 classes.dex 文件**

**2、使用 dex2jar 将 classes.dex 转成 jar 文件**

**3、将 jar 文件用 jd-gui 打开，查看源代码**

**4、adb定位到类名包路径，找到相关代码**

**5、apktool 反编译 apk，找到 smali 对应的爆破点**

**6、修改 smali 文件，调试程序**

**7、重新打包，重新签名**

以上是我对这次破解流程的一个总结，如果有不对或者遗漏的地方，还请各位大佬指正。

最近有点感冒，干咳一个多月了还不好起来，想到小菊花妈妈课堂那句话：**孩子咳嗽老不好，多半是废了**。我也就放弃治疗，待在空荡的房间，干着喜欢干的事^_^。我也是刚接触Android逆向没多久，一开始以为很复杂，很麻烦，当时只是抱着无聊想试试的心态，反正都放弃治疗了，没想到只花了一个多小时，竟然就成功了，并没有想象中的那么难。

如果你对逆向也感兴趣的话，并且和我一样是初学者，我觉得这个实战过程非常适合你。一是能让你感受到破解的整个过程；二是难度不大，不会打击到你的兴趣，同时还能得到一定的成就感。

## License

    Copyright 2019 goldze(曾宪泽)
 
    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at
 
        http://www.apache.org/licenses/LICENSE-2.0
 
    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.