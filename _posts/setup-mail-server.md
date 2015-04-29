title: "搭建的postfix+dovecot邮件服务器"
date: 2015-04-26 21:40:07
tags: [maintenance, cheatsheet, gnus]
---

# 配置postfix+dovecot

[Linux Mail Server Setup](http://www.linuxmail.info/)上的配置过程已经很详细了，按照How to install SMTP, POP3, IMAP and Webmail一节里几个page的步骤做下来，一个基本的mail server基本就起来了。

不过因为年代久远（2010年的产物），还有些地方需要修改和完善。

## 配置smtpd relay restriction

这是main.cf里对转发权限的配置项，在linuxmail上用的是smtpd\_recipient\_restrictions，但是到了2.10+的postfix里需要改成smtpd\_relay\_restrictions，也就是：

    smtpd_relay_restrictions =
	    permit_mynetworks
		permit_sasl_authenticated
		reject_unauth_destination

## 配置SSL/TLS

这个过程没被列在首页：http://www.linuxmail.info/postfix-dovecot-ssl/ 。

要开启smtp over SSL/TLS，还需要把/etc/postfix/master.cf里下面这几行的注释去掉：

    smtps     inet  n       -       n       -       -       smtpd
     -o syslog_name=postfix/smtps
     -o smtpd_tls_wrappermode=yes
     -o smtpd_sasl_auth_enable=yes
     -o smtpd_reject_unlisted_recipient=no
     -o smtpd_client_restrictions=$mua_client_restrictions
     -o smtpd_helo_restrictions=$mua_helo_restrictions
     -o smtpd_sender_restrictions=$mua_sender_restrictions
     -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
     -o milter_macro_daemon_name=ORIGINATING

Dovecot的SSL配置在dovecot官方wiki上有：[SSL/Dovecot SSL configuration](http://wiki2.dovecot.org/SSL/DovecotConfiguration)。

# Apache httpd使能https

CentOS里的squirrelmail默认会把http来的请求重定向到https，所以使能https就是必须的了。CentOS官方wiki就有详细步骤：[HowTos/Https](http://wiki.centos.org/HowTos/Https)。

# 查看邮件服务器域名的解析

工具自然是nslookup，但是没有找到命令行参数的办法，只好用交互式：

    $ nslookup
    > set type=mx
    > your_mail_addr.your_mail_domain

# 用telnet/openssl测试邮件服务器

## SMTP

连接服务器：

    telnet <server> 25
	openssl s_client -crlf -connect <server>:465

### 服务器状态

    > ehlo <hostname>
	250-xxxxxx
	250-PIPELINING
	250-SIZE 10240000
	250-VRFY
	250-ETRN
	250-AUTH PLAIN LOGIN
	250-AUTH=PLAIN LOGIN
	250-ENHANCEDSTATUSCODES
	250-8BITMIME
	250 DSN

### 账号验证

    > auth plain ###

其中###就是用户名和密码串在一起之后的base64编码。

### 发送邮件

    > mail from: <user>
	250 2.1.0 Ok
	> rcpt to: <test@example.org>
	250 2.1.5 Ok
	> data
	354 End data with <CR><LF>.<CR><LF>
	> test
	> .
	250 2.0.0 Ok: queued as 9729067C17

如果rcpt to之后报454: Relay access denied，一般是服务器要求发送非本地邮件时需要验证账号，在mail from之前用auth plain ###登录即可。

### 退出

    > quit

## IMAP

连接服务器：

    telnet <server> 143
	openssl s_client -crlf -connect <server>:993

以下是常用的imap命令。
* 注1：每行前面的a是个tag，每一条imap命令前都需要有一个，可以随意取）；
* 注2：命令大小写不敏感。

### 登录

    > a login user password
    a OK [CAPABILITY IMAP4rev1 LITERAL+ SASL-IR LOGIN-REFERRALS ID ENABLE IDLE
	SORT SORT=DISPLAY THREAD=REFERENCES THREAD=REFS THREAD=ORDEREDSUBJECT
	MULTIAPPEND URL-PARTIAL CATENATE UNSELECT CHILDREN NAMESPACE UIDPLUS
	LIST-EXTENDED I18NLEVEL=1 CONDSTORE QRESYNC ESEARCH ESORT SEARCHRES WITHIN
	CONTEXT=SEARCH LIST-STATUS SPECIAL-USE BINARY MOVE] Logged in

user不需要@和后面的domain name（至少我自己搭的Dovecot是这样）。

### 列举邮箱目录

    > b LIST "" "*"

### 查看目录状态

    > c SELECT INBOX

### 退出

    > d LOGOUT

# + Emacs Gnus?!

一切正常的话，Gnus的配置没有什么特别的（毕竟还是接一个标准的mail server嘛），仍然是gnus-select-method配置nnimap，message-send-mail-function用smtpmail-send-it，最后加上一些服务器的域名端口之类就算大功告成，当然这是“一切正常的话”……

## gnutls-cli卡死

反正msys2是不太正常，mingw包里的gnutls-cli和openssl各种不认stdin，结果就是尝试SSL连接之后emacs就万年不动等timeout了，暂时也没工夫去帮他们debug & fix。好在msys里的openssl尚可一用，所以在emacs里多配一句：

    (setq tls-program '("path_to_openssl s_client -connect %h:%p"))

就能暂时workaround过去。

## smtpmail-send-it: Send failed: RENEGOTIATING

查了一下RENEGOTIATING，好像还是个臭名昭著的CVE漏洞，所以一开始还以为gnus已经智能到会自动探测server有没有开renegotiating，并且拒绝通过这种server发送邮件的程度。不过最后看下来，这个bug跟CVE半毛钱关系都没有啊~~~ 其实root cause是：

* smtp里指定收件地址用的是rcpt to；
* openssl s_client里可以用R触发RENEGOTIATING
* 注意smtp本身对大小写不敏感，所以……如果client通过openssl发送RCPT TO的话……

解决方法倒是简单，找/share/emacs/24.5/lisp/mail/smtpmail.elc（没错，就是elc，懒得改gz里的源码再byte compile了），找到RCPT TO，把这几个字符全改成小写，搞定！

# References

[Setting up SMTP Authentication over TLS with Postfix](http://rene.bz/setting-smtp-authentication-over-tls-postfix/)
[Access IMAP server from the command line using OpenSSL](https://delog.wordpress.com/2011/05/10/access-imap-server-from-the-command-line-using-openssl/)
[Installing and configuring Spamassassin on CentOS](http://www.rackspace.com/knowledge_center/article/installing-and-configuring-spamassassin-on-centos)
