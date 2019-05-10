---
title: Java与Shell的交互-敏感信息传递
date: 2018-09-16 15:48:51
subtitle: java&shell
tags:
- java
- shell
---

## 简介

1. shell解析json (好像是jq)会有安全问题,要求通过java或python进行装换;
2. java与shell交互时,不能通过参数传递敏感信息（密码、证件号等）,会导致信息泄露(查看进程时明文展示) ,可通过环境变量传递。

## 用例

shell传java:

```shell
export temp_instance_pwd=${temp_instance_pwd};
```

java获取：

```java
System.getenv("temp_instance_pwd");
```



java传shell:

```java
    ProcessBuilder procBuilder = new ProcessBuilder(args).directory(newFile(System.getProperty("user.dir")));
    Map<String, String> env = procBuilder.environment();
    env.put("temp_instance_pwd", MigrateProperties.getTmpInstancePwd());
    @Cleanup("destroy")
    Process proc = procBuilder.start();
```

shell获取：

```shell
pwd=${temp_instance_pwd}
```

