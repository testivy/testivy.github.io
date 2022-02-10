---
title: 使用semgrep 分析log4j2到放弃
commentable: true
Edit: 2018-11-14
mathjax: true
mermaid: true
tags: log4j semgrep
categories: 漏洞分析
description: 使用semgrep看看能否找到Sink和Source。
---

看到很多师傅都是使用codeql 来找log4j2 shell 的Sink和Source，例如FreeBuf 上的文章《codeql分析log4j 》https://www.freebuf.com/articles/web/318141.html 、先知社区《利用CodeQL分析并挖掘Log4j漏洞》https://xz.aliyun.com/t/10707 。但是好像没看到有人使用semgrep 来分析log4j的，因此就来尝试使用semgrep看看能否很容易地找到Sink和Source。

题外话：发现semgrep 网站上也有了专门针对log4j2 shell 漏洞检测规则，https://semgrep.dev/editor?registry=java.log4j.security.log4j-message-lookup-injection.log4j-message-lookup-injection 。怎么说呢，这个规则可以用于前期应急的时候代码检测，但是后期就没办法知道哪些升级了哪些没升级，是否用的是安全版本了。

![img](https://cdn.nlark.com/yuque/0/2022/png/2022318/1643442880867-a43ced08-2f91-414f-8e90-bfe9dc82ddb7.png)
## 环境搭建
相比codeql 需要对log4j源码进行编译生成代码数据库的过程，semgrep 不需要对需要分析的代码进行编译的，因此直接把logging-log4j2-rel-2.14.1 这个版本的源码下载到本地即可，非常简单。

这里我使用semgrep  docker 方式部署来进行代码扫描的：
```bash
docker run --rm -v "${PWD}:/src" returntocorp/semgrep --config=s/xx  ./  -o scan-c.txt  --time --debug
```
关于semgrep 的更多详细使用方式可参考我之前的文章 [semgrep SAST编写自定义漏洞检测规则实践](https://www.freebuf.com/articles/web/286643.html)
## 编写规则
当然Sink 点还是落在JNDI注入的lookup方法上，所以首先初步匹配下lookup 方法，看看能匹配出多少个lookup。
只使用一个pattern，规则也是比较容易就写出来的:
```yaml
rules:
- id: log4j-lookup
  pattern:
        $NAME.lookup(...);
  message: |
      find lookups
  languages: [java]
  severity: WARNING
  metadata: 
      category: security
      technology:
        - java
      confidence: MEDIUM
```
在调试阶段，我们可以在[https://semgrep.dev/editor](https://semgrep.dev/editor) 上编写规则，验证好了之后可以通过share方式，然后通过--config=  去加载指定的规则。

![img](https://cdn.nlark.com/yuque/0/2022/png/2022318/1643438750979-d7e750fe-1643-4473-b598-c402c3b16916.png)
![img](https://cdn.nlark.com/yuque/0/2022/png/2022318/1643438515852-dea96223-81b6-4480-b1bf-0b82cf213cd6.png)
从上图可以看到，找到了16处存在调用lookup方法的代码行位置:
![img](https://cdn.nlark.com/yuque/0/2022/png/2022318/1643439608024-6a06e13b-1a77-410b-b74e-4b5136ce7e80.png)
### 优化规则

上一步的规则找到了16处的问题，但是一个一个去分析有点多，有必要进一步筛选下。知道JNDI注入会使用JNDI context方式，其中应该都会引入javax.naming.Context 或者 javax.naming.InitialContext 类，而且InitialContext 实现了Context 接口。
因此继续改造下规则，这次我们引入patterns 关键字来进行限定：
- (1) 要求调用lookup方法前导入了 <u>javax.naming.Context</u>
- (2) 要求调用lookup方法前导入了 <u>javax.naming.InitialContext</u>
基于上面的两种情况，编写如下规则：
