---
title: 使用semgrep 分析log4j2到放弃
commentable: true
Edit: 2018-11-14
mathjax: true
mermaid: true
tags: log4j 漏洞分析
categories: 漏洞分析
description: 使用semgrep看看能否找到Sink和Source。
---

看到很多师傅都是使用codeql 来找log4j2 shell 的Sink和Source，例如FreeBuf 上的文章《codeql分析log4j 》https://www.freebuf.com/articles/web/318141.html 、先知社区《利用CodeQL分析并挖掘Log4j漏洞》https://xz.aliyun.com/t/10707 。但是好像没看到有人使用semgrep 来分析log4j的，因此就来尝试使用semgrep看看能否很容易地找到Sink和Source。

题外话：发现semgrep 网站上也有了专门针对log4j2 shell 漏洞检测规则，https://semgrep.dev/editor?registry=java.log4j.security.log4j-message-lookup-injection.log4j-message-lookup-injection 。怎么说呢，这个规则可以用于前期应急的时候代码检测，但是后期就没办法知道哪些升级了哪些没升级，是否用的是安全版本了。

