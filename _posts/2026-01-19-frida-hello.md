---
title: "Android frida使用记录：一次排查国密组件多线程问题的经历"
date: 2026-01-19 19:00:00 -0000
tags: frida android BouncyCastle 动态分析
---

本文记录了一次在使用frida进行Android应用动态分析时的操作步骤和心得体会，通过一个在线上遗留了有好几年的问题的解决，分享给大家。

<!--more-->

# 问题的背景

目前手上维护的一个android app，在使用国密组件的时候，有万一二的概率会出现`SM3 MessageDigest not available` 和 `java.security.NoSuchProviderException No such provider: BC`的错误日志监控，由于出现的概率比较低，且内部做了自动降级，对线上用户没有任何的影响，一直没有重视。

最近发了一个新版本，只是给另一个模块的http接口的调用时机提前了，导致这个问题出现的概率大幅提升，需要排查解决。

# 初步排查

通过线上用户日志的初步分析，有如下现象：

1、用户进入模块A，国密组件正常；
2、在模块A调用了B模块的http接口，B模块的http接口正常；
3、如果此时，A模块的Http组件响应过慢，则A模块在使用国密组件的时候，就会出现题中的错误信息。

通过对`No such provider: BC`日志的初步分析，确定，一定是在什么时机，BouncyCastle的Provider被remove了。此时，需要验证是否有其他线程在操作Security Providers，线上app的修改比较困难（需要构建和加固，而加固是需要收钱的），所以决定使用frida进行动态分析。

# 使用frida排查

在使用Frida时，首先需要对手机进行root（虽然不root 也能行，但是在我当前的诉求下，root是一个更优的选择），手上有一台MI8，之前刷了LineageOS(LinearageOS 22.2-20251129-NIGHTLY-dipper)，在开发者选项里面直接就有root的选项(设置→系统→开发者选项→Root身份的调试)，打开即可。

# 安装frida-server

跟着[官网文档](https://frida.re/docs/examples/android/)，下载对应的fria-server即可，此时下载的最新版本是
[17.6.0](https://github.com/frida/frida/releases/tag/17.6.0)，下载好之后直接解压，然后通过adb push到手机上：

```bash
# 以root身份运行shell
adb root
adb push frida-server-17.6.0-android-arm64 /data/local/tmp
adb shell
cd /data/local/tmp
chmod 755 frida-server-17.6.0-android-arm64
./frida-server-17.6.0-android-arm64 &
```

# 安装frida-python脚本

在本地机器上安装frida的运行脚本，官方推荐使用Python pip 来安装，在国内会有下载慢和一些其他网络问题，这里可直接使用阿里云的镜像，下载速度贼快，如下

```bash
pip install frida-tools -i https://mirrors.aliyun.com/pypi/simple/
```

安装完成后，可使用测试脚本`frida-ps -U`，来查看手机上运行的进程列表,如果能看到如下的输出，说明基本上环境搭建成功。

```bash
PID  Name
---------------------------
1234 com.example.app
5678 system_server
...
```

其他frida的使用文档可参考[官方文档](https://frida.re/docs/android/)。

## 注意

如果遇到`Failed to spawn: need Gadget to attach on jailed Android` 类似的错误，如下，可以以root的身份重新运行一下frida-server，即可。

```bash
➜  frida frida -U -f com.netease.epay -l frida-hook-security-providers.js
     ____
    / _  |   Frida 17.6.0 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to MI 8 (id=16fc1018)
Failed to spawn: need Gadget to attach on jailed Android; its default location is: /Users/nky/.cache/frida/gadget-android-arm64.so
```

正常的日志如下：

```bash
➜  frida frida -U -f com.netease.epay -l frida-hook-security-providers.js
     ____
    / _  |   Frida 17.6.0 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to MI 8 (id=16fc1018)
Spawned `com.netease.epay`. Resuming main thread!
```

# 编写hook脚本

本次需要hook的类是`java.security.Security`，主要是`removeProvider`方法，[脚本](https://gist.github.com/nky89/947161199fd486dfcc0ebae2e5035233)内容如下：

```javascript
// frida-hook-security-providers.js
Java.perform(function () {
  var Security = Java.use("java.security.Security");
  var ExceptionCls = Java.use("java.lang.Exception");
  var Log = Java.use("android.util.Log");

  function dumpStack() {
    try {
      var e = ExceptionCls.$new("trace");
      return Log.getStackTraceString(e);
    } catch (err) {
      return "stack-unavailable: " + err;
    }
  }

  function log(tag, msg) {
    try {
      Log.i(tag, msg);
    } catch (e) {
      console.log(tag + ": " + msg);
    }
  }

  Security.removeProvider.overload("java.lang.String").implementation =
    function (name) {
      var info = 'removeProvider("' + name + '") called\\n' + dumpStack();
      log("SEC_PROV", info);
      return this.removeProvider(name);
    };

  Security.insertProviderAt.overload(
    "java.security.Provider",
    "int",
  ).implementation = function (provider, pos) {
    var pname = "null";
    try {
      pname = provider ? provider.getName() : "null";
    } catch (e) {}
    var info =
      "insertProviderAt(provider=" +
      pname +
      ", pos=" +
      pos +
      ") called\\n" +
      dumpStack();
    log("SEC_PROV", info);
    return this.insertProviderAt(provider, pos);
  };

  Security.addProvider.overload("java.security.Provider").implementation =
    function (provider) {
      var pname = "null";
      try {
        pname = provider ? provider.getName() : "null";
      } catch (e) {}
      var info = "addProvider(provider=" + pname + ") called\\n" + dumpStack();
      log("SEC_PROV", info);
      return this.addProvider(provider);
    };

  Security.getProvider.overload("java.lang.String").implementation = function (
    name,
  ) {
    var p = this.getProvider(name);
    var pname = p ? (p.getName ? p.getName() : "obj") : "null";
    log("SEC_PROV", 'getProvider(\"' + name + '\") -> ' + pname);
    return p;
  };

  var MD = Java.use("java.security.MessageDigest");
  MD.getInstance.overload(
    "java.lang.String",
    "java.lang.String",
  ).implementation = function (alg, providerName) {
    try {
      return this.getInstance(alg, providerName);
    } catch (e) {
      log(
        "SEC_PROV",
        'MessageDigest.getInstance(\"' +
          alg +
          '\", \"' +
          providerName +
          '\") threw: ' +
          e +
          "\\n" +
          dumpStack(),
      );
      throw e;
    }
  };

  log("SEC_PROV", "security provider hooks installed");
});
```

# 启动并分析日志

使用如下命令启动app并加载脚本：

```bash
frida -U -f com.netease.epay -l frida-hook-security-providers.js
```

然后查看日志，可以看到明显的remove 和add，已经调用的时差序：

```
16:14:59.289 SEC_PROV                 I  getProvider("BC") -> BC
16:14:59.291 SEC_PROV                 I  getProvider("BC") -> BC
16:14:59.488 SEC_PROV                 I  removeProvider("BC") called\njava.lang.Exception: trace
                                         	at java.security.Security.removeProvider(Native Method)
```

## 说明

上面的日志，移除了敏感信息，实际上是有调用的完整堆栈，方便分析。

# 定位原因

有了上面清晰的调用堆栈，很快就能定位到多线程操作Security Providers的问题：另外一个组件在初始化的时候先remove，然后再添加。remove-add 的时间窗口内，整个系统是没有BC的，而A组件在这个时间窗口又使用BC组件导致了题目中的问题，因此解决方案也就不难了，根据项目中的代码调整即可。

# 总结

通过这次使用frida进行动态分析，成功定位并解决了一个线上遗留已久的多线程问题，整个过程还是比较顺利的。firda对于这种不能修改代码（特别是系统方法的调用），不能对线上添加日志的场景，提供了一个非常好的解决方案。希望本文能对大家有所帮助。
