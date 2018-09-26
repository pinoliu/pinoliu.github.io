---
title: Java进程调用系统命令-ProcessBuilder与Runtime
date: 2018-09-16 11:14:48
subtitle: processbuilder&runtime
tags:
- java
- thread
- shell
---

## 简介

ProcessBuilder.start()和Runtime.exec()方法都被用来创建一个操作系统进程(执行命令行操作) ,并返回Process子类的一个实例,该实例可用来控制进程状态并获得相关信息。

ProcessBuilder.start()和Runtime.exec()传递的参数有所不同。Runtime.exec()可接受一个单独的字符串,这个字符串是通过空格来分隔可执行命令程序和参数的，也可以接受字符串数组参数。而ProcessBuilder的构造函数是一个字符串列表或者数组，列表中第一个参数是可执行命令程序,其他的是命令行执行是需要的参数。

ProcessBuilder相对于Runtime提供了对进程更多的控制，是 JDK 1.5后才加入的。而Runtime是从 JDK 1.0 加入，后续多个版本重构。

通过查看JDK源码可知, Runtime.exec最终是通过调用ProcessBuilder来真正执行操作的。

推荐：使用ProcessBuilder调用系统命令。

注意：及时取走标注输出和错误输出的输出流，避免占用系统资源。

## 代码用例

```java
    ProcessBuilder procBuilder =
        new ProcessBuilder(args).directory(newFile(System.getProperty("user.dir")));
    // 标准输出和标准错误,统一合并到Process.getInputStream()输出流
    procBuilder.redirectErrorStream(true);
    // 设置环境变量
    Map<String, String> env = procBuilder.environment();
    // env.put("temp_instance_pwd", MigrateProperties.getTmplnstancePwd();
    @Cleanup("destroy")
    Process proc = procBuilder.start();
    @Cleanup
    BufferedReader in = new BufferedReader(newInputStreamReader(proc.getInputStream(), "UTF-8"));
    String echoStr = null;
    StringBuilder inputStrBuilder = new StringBuilder();
    while ((echoStr = in.readLine()) != null) {
      inputStrBuilder.append("rn");
      inputStrBuilder.append(echoStr);
    }
    if (StringUtils.isNoneEmpty(inputStrBuilder.toString())) {
      LOG.info("RunCmdUtil processResult echo warn: 0", inputStrBuilder);
    }

    int ret = proc.waitFor();
    LOG.info("RunCmdUtil processResult: 0", ret);
```



## 参考

https://blog.csdn.net/c315838651/article/details/72085739

https://www.cnblogs.com/taven/archive/2011/12/17/2291460.html

http://desert3.iteye.com/blog/1596020