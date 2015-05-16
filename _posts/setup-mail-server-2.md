title: "搭建的postfix+dovecot邮件服务器（2）：SPF与IP Reputation"
date: 2015-05-16 10:27:38
category: Techniques
tags: [maintenance, cheatsheet]
---

![SPF Logo](spf-logo.png)

（图片来源：http://www.openspf.org/Logo）

最近发到outlook.com和live.com的邮件被各种退回，理由是550 OU-002（与内容包含垃圾邮件特征或 IP 地址/域信誉相关……⊙_⊙）查了一下，可能和SPF有关系，于是匆忙补课……

# SPF（Sender Policy Framework）

SPF是以DNS TXT record（也只能是TXT record）的形式保存于域名的DNS信息，用于验证邮件FROM域的地址与发送邮件的服务器IP是否一致，以避免他人伪造邮件地址。由于SMTP协议本身并没有定义任何检查FROM域真实性的手段，FROM域随便填一个的无所谓，所以伪造邮件发送地址，用机器（例如sendmail命令）群发大量邮件是一件很稀松平常的事情（我就干过……当然是出于正当原因……）。SPF就是用于弥补SMTP这方面的不足。

# DNS使能SPF

SPF策略保存在域名的TXT record中，而一条TXT record本身就是一个字符串，所以使能SPF就是如何按照SPF的格式要求编写策略的问题。

以下是从RFC里拿来的SPF策略的语法（原文用的是RFC 5234定义的ABNF，这里我就用我习惯的RE了）：

    <name>             := [:alpha:] ([:alnum:] | "-" | "_" | ".")*

    <qualifier>        := "+" | "-" | "?" | "~"
	<mechanism>        := <all> | <include> | <a> | <mx> | <ptr> | <ip4> | <ip6> | <exists>
	<directive>        := <qualifier>? <mechanism>

	<unknown-modifier> := <name> "=" <macro-string>
	<modifier>         := <redirect> | <explanation> | <unknown-modifier>

	<terms>            := ([:space:]+ (<directive> | <modifier>))*

    <version>          := "v=spf1"

    <policy>           := <version> <terms> [:space:]*

简单地说，一个SPF“策略”是以“v=spf1”开头，后面跟一堆term的字符串，每个term可以是directive（决定验证结果）或者modifier（不直接决定验证结果，这里不详细说）。一个验证过程有三个输入，包括邮件发送方的IP（<ip>）、发件邮箱的域名部分（<domain>）和完整的FROM域（<sender>），输出的是以下验证结果中的一个：

* None：无法验证，可能因为FROM域找不出一个域名，或者DNS里没有有效的SPF信息
* Neutral：不确定是否有效（对应的qualifier是“?”）
* Pass：地址确认有效（对应“+”）
* Fail：地址确认无效（对应“-”）
* Softfail：地址很可能无效（对应“~”）

还有一些其它可能的返回情况，诸如Temperror、Permerror等，表示验证过程中发生了异常。

验证过程很直接，从前往后一条一条地看TXT里的SPF策略，尝试匹配每一个directive term里的mechanism，匹配上了就返回qualifier指定的结果（默认为Pass）。一些常用的mechanism的详细定义如下（具体参见RFC7208第5节）：

* all：就一个字符串"all"，匹配总是成功，一般用来指定默认结果，例如"-all"；
* include：向指定域名发起递归的验证请求，如果该域名认为地址有效，则匹配成功，例如"include:example.org"；实际配置中大量使用这种mechanism来组织各种policy；
* a：如果<ip>在指定域名的A或AAAA记录中出现，则匹配成功，例如"a:example.org"；RFC里明写了":"后面的域名可以省略，但是没说默认值，估计就是<domain>吧；
* mx：和a差不多，区别在于这里查的是MX记录；
* ip4：如果<ip>是IPv4地址，且指定mechanism指定的子网里，则匹配成功，例如："ip4:192.168.0.112"（相当于"ip4:192.168.0.112/32"）或者"ip4:192.168.0.0/24"；
* ip6：匹配IPv6地址，形式与ip4相同；
* ptr：（RFC里明确表示不建议使用，只是为了兼容已有SPF策略才保留这一项）

这么看来，对于个人邮件服务器来说，最简单有效的SPF策略应该就是："v=spf1 a mx -all"

# 查验SPF配置

既然本质上是DNS TXT record，那自然是用nslookup：

    $ nslookup -q=txt mail.server.domain

有一些在线工具可以进一步查验SPF是否有效，例如：[Sender Policy Framework](http://mxtoolbox.com/spf.aspx)可以给出SPF记录的parse结果，Kitterman的[测试工具](http://www.kitterman.com/spf/validate.html)可以验证SPF是否如期工作。

一件好玩的事情是看看各大ISP的SPF配置：

* Gmail：在gmail.com有SPF记录，指向_spf.google.com，_spf.google.com又指向_netblocks.google.com，_netblocks2.google.com和_netblocks3.google.com，_netblocks.google.com里面填了各种IPv4子网，_netblocks2.google.com里面填了各种IPv6子网，_netblocks3.google.com拒绝一切；
* Hotmail和Live：在hotmail.com和live.com能找到类似的SPF记录，include的数量比gmail多了好多，没一个个往下查……
* Yahoo：在yahoo.com能找到SPF记录，一样只include了另一个域名_spf.mail.yahoo.com，然后这里出现了ptr……
* 163, 126, Yeah：在163.com，126.com和yeah.net能找到相同的SPF记录，都include了spf.163.com，spf.163.com又include了[abcd].163.com，前三个填了一堆IPv4子网，第四个干脆连DNS记录都没有……
* QQ：在qq.com能找到一条include记录指向spf.mail.qq.com，spf.mail.qq.com里面include了spf-[abcd].mail.qq.com，四个域名里是各不相同的一堆IPv4子网。

看起来各大ISP在SPF的配置上都差不多，用include把几个域名组织成树状结构，树根是邮箱的域名，各自的IP地址范围则铺陈在各个叶节点。

BTW，名字怎么都是abcd和123……

# 邮件服务器验证SPF

验证SPF是spam checker的事情。在spamassassin里默认已经使能了SPF验证（/etc/mail/spamassassin/init.pre里load了SPF的Plugin），不需要额外做什么配置（因为这货连一个配置选项都没有……）。

# 关于IP Reputation

ESP（Email Service Provider）圈子里对一个发件人有多大可能性发出垃圾邮件是有评价的，总的来说叫做IP Reputation。和人的Reputation一样，一个IP作为一个mail sender的Reputation取决于它先前做过什么事情：有没有给不存在的地址发过信？它发的邮件有没有被别人标记成“垃圾”？有没有人根据这个地址拒收邮件？等等等等……有些互联网安全公司将提供IP Reputation信息作为一种服务对外销售，例如Symantec（赛门铁克）。

而SPF作为一种对域名的“身份认证”，也是在评价IP Reputation的考查范围之内。

一些ESP会根据IP Reputation对某些地址采取blacklist之类的操作。前一阵子发往hotmail的邮件都被莫名其妙地退回来，今天在微软的[Sender Information Form](https://support.microsoft.com/en-us/getsupport?oaspworkflow=start_1.0.0.0&wfname=capsub&productkey=edfsmsbl3&ccsid=635670352520199342)上提交了复查申请之后，才知道是IP被Symantec's BrightMail filter拉黑了，去[Symantec](http://ipremoval.sms.symantec.com/lookup/)上一查，还真有一条negative reputation，说：

    The host is unauthorized to send email directly to email servers

说白了，就是你得“证明”你是个mail server，果然就是SPF的问题……

# 总结

SPF是一个被普遍使用的一种强化邮件服务器域名“合法性”的协议，对于个人邮件服务器来说，应该配置一下这个东西（也就一句话的事情而已），以免自己的服务器被莫名其妙地拉黑，落到发不了邮件的地步……

# Reference

[Sender Policy Framework (SPF) for Authorizing Use of Domains in Email, Version 1](http://tools.ietf.org/html/rfc7208)
[Mail::SpamAssassin::Plugin::SPF - perform SPF verification tests](http://spamassassin.apache.org/full/3.0.x/dist/doc/Mail_SpamAssassin_Plugin_SPF.html)
[IP Reputation](http://wiki.wordtothewise.com/IP_Reputation)
