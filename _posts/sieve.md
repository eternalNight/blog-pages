title: "postfix+dovecot上配置sieve"
date: 2015-04-29 20:08:45
tags: [maintenance, cheatsheet, gnus]
---

![Sieve的Logo（取自sieve.info）](logo.png)

# sieve是什么鬼？

简单来说，Sieve就是个用于定义邮件过滤器的语言，作用和procmail的配置文件类似，算是
后者的替代品。目前常用的IMAP服务器cyrus和dovecot都支持Sieve（Sieve就是在cyrus项目
中被提出来的）。

除了语言本身（RFC 5228）以外，Sieve还伴有一个用于在客户端管理服务端Sieve脚本的协
议ManageSieve（RFC 5804），同样都被cyrus和dovecot支持。

# 在Postfix+Dovecot中配置Sieve和ManageSieve

首先让postfix收到目标为本机的邮件之后，交给dovecot进行投递。在
/etc/postfix/master.cf里：

    mailbox_command = /usr/libexec/dovecot/deliver

然后是在dovecot中使能处理Sieve脚本的插件pigeonhole，在
/etc/dovecot/conf.d/15-lda.conf里：

    protocol lda {
        mail_plugins = $mail_plugins sieve
	}

在/etc/dovecot/conf.d/20-ltmp.conf里：

    protocol ltmp {
        mail_plugins = $mail_plugins sieve
	}

然后是指定sieve脚本的位置，在/etc/dovecot/conf.d/90-sieve.conf里：

    plugin {
	    sieve = file:~/.sieve
		sieve_dir = ~/.sieve.d
	}

其中sieve一项是真正起作用的Sieve脚本，简单起见这里就指定了每个用户HOME目录下的一
个文件，在ManageSieve使能之后，这会变成一个指向sieve_dir下某个文件的符号链接。
sieve_dir是ManageSieve用于保存所有上传Sieve脚本的目录。

最后使能ManageSieve，在/etc/dovecot/conf.d/20-managesieve.conf里：

    protocols = $protocols sieve
	service managesieve-login {
	    inet_listener sieve {
		    port = 4190
			ssl = yes
		}
	}

ManageSieve默认使用4190端口，这里将ManageSieve service架在了SSL/TLS之上。

ManageSieve协议同样可以用telnet和openssl测试，Dovecot的wiki上给出了基本步骤：
http://wiki2.dovecot.org/Pigeonhole/ManageSieve/Troubleshooting 。

# Sieve脚本编写

Cheatsheet time！

## 总体框架

    require ["fileinto", "reject"];
	if ... {
	    ...
	} elsif ... {
	    ...
	}
	if ... {
	    ...
	} else {
	    ...
	}

Sieve语言设计了一种扩展机制，允许在脚本中用require ...的方式使用各种扩展提供的功
能。一个Sieve服务器可用的扩展可以用telnet/openssl连上去查看，例如：

    "SIEVE" "fileinto reject envelope encoded-character vacation subaddress
    comparator-i;ascii-numeric relational regex imap4flags copy include
    variables body enotify environment mailbox date ihave"

余下的部分是一大串if...then...else...语句，和C一样，if/then后跟的是对邮件某些属性
的测试，大括号内的则是满足相应测试的邮件改如何处理，一个例子是：

    if header :contains "from" "linkedin.com" {
        fileinto "notification.linkedin";
	}

上面的脚本会把所有发件人地址包含linkedin.com的邮件放到notification.linkedin目录下。

## 邮件属性测试

| 测试语句           | 含义              |
|:------------------:|:-----------------:|
| header :contains ["to", "cc"] "xxx" | 邮件头的to或cc域包含xxx |
| not <test> | 如果<test>不成立 |

## 邮件处理

| 处理语句           | 含义              |
|:------------------:|:-----------------:|
| fileinto "xxx"; | 将邮件投入xxx文件夹中 |
| discard; | 丢弃邮件 |
| redirect "xxx@xxx.net"; | 将邮件转发给xxx@xxx.net |
| keep; | 将邮件保存到默认文件夹中 |

# +Gnus

Emacsen的必备工作，就是要把所有事情都放到Emacs里来做，Sieve当然不能例外……

## 配置sieve-manage

Gnus有编辑和管理Sieve脚本的能力，当然还是需要一些额外配置和调整，例如：

* 配置ManageSieve服务端口，对应变量为sieve-manage-default-port，其默认值是sieve，
  也就是找系统中/etc/services文件里sieve一项对应的端口号，Linux下一般没问题，但
  Windows下的services（即Windows/System32/Drivers/etc/services）里没有sieve一项，
  所以对于windows下的emacsen来说，要么设置sieve-manage-default-port为4190，要么在
  windows的services里把sieve加进去。
* 配置ManageSieve的连接类型，对应变量为sieve-manage-default-stream，默认值
  是'network，也就是一般的TCP连接。如果4190是架在SSL/TLS上的话，就需要将
  sieve-manage-default-stream设为'tls（或'ssl，效果是一样的）。
* 配置ManageSieve服务端的行尾，对应变量为sieve-manage-server-eol，默认是windows式
  的"\r\n"，需要改成linux式的"\n"。

上面的变量都通过custom-set-variables设置，也就是：

    (custom-set-variables
     '(sieve-manage-default-port 4190)
     '(sieve-manage-default-stream 'tls)
     '(sieve-manage-server-eol "\n"))

当然还有用于SSL/TLS连接工具的问题，windows/msys2下的诸位得设一下通用的tls-program：

    (custom-set-variables
     '(tls-program '("/path/to/msys/openssl.exe s_client -connect %h:%p")))

注意不需要-crlf，不然Gnus会报错。

## Sieve编辑模式的快捷键

cheatsheet time again！

|快捷键               |功能           |
|:-------------------:|:-------------:|
| C-c RET | 建立sieve连接 |
| C-c C-l | 将当前buffer作为sieve脚本，通过已建立的连接上传到服务器 |

## Gnus管理Group的快捷键

Group是Gnus里对应于服务器上Folder的东西。有了邮件过滤之后，跑不了要在Gnus里做一点
Group管理的事情。

|快捷键               |功能           |
|:-------------------:|:-------------:|
|G m|新建一个Group|
|G r| 重命名当前Group|
|G del| 删除当前Group |
|B m | 移动当前邮件到另一Group |
|B del|删除当前邮件|
|\#|：标记当前邮件|
|M-\#|取消当前邮件的\#标记|

# References

[Sieve](http://sieve.info/)
[Pigeonhole Sieve support](http://wiki2.dovecot.org/Pigeonhole)
