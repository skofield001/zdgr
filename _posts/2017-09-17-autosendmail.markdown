---
layout: post
category: "matlab"
title:  "每日自动检查文本文件并邮件通知"
---
<br/>

<p>实习期间一项日常工作是记录每日各位老师使用公共饭卡情况，经常出现有实习生当日漏记的现象，为此我捡起以前写脚本的老本行，写了个dos批处理脚本每天检测，安装了一个sendmail程序，通过windows定时任务每天将检测结果发送到邮箱</p>
<p>主程序脚本内容如下：</p>

<!-- more -->
```
@ECHO OFF
REM send email from command line via SMTP with sendmail
set Today=%date:~0,4%%date:~5,2%%date:~8,2%
dir S:\实习生常用表格\饭卡记账.txt|findstr "饭卡记账.txt" > S:\朱俊禹\每日饭卡记账\%Today%_env.txt
set /p a=< S:\朱俊禹\每日饭卡记账\%Today%_env.txt
set updatetime=%a:~0,4%%a:~5,2%%a:~8,2%

ECHO From: phantom2026@163.com > S:\朱俊禹\每日饭卡记账\%Today%_mail.txt
ECHO To: zhujunyu@cpic.com.cn >> S:\朱俊禹\每日饭卡记账\%Today%_mail.txt
if %updatetime%==%Today% (ECHO Subject: %Today%饭卡记账 >> S:\朱俊禹\每日饭卡记账\%Today%_mail.txt) else (ECHO Subject: %Today%饭卡记账未更新！！！ >> S:\朱俊禹\每日饭卡记账\%Today%_mail.txt) 
ECHO.>> S:\朱俊禹\每日饭卡记账\%Today%_mail.txt
more S:\实习生常用表格\饭卡记账.txt|findstr %Today% >> S:\朱俊禹\每日饭卡记账\%Today%_mail.txt
ECHO 以上为%Today%饭卡记账情况 >> S:\朱俊禹\每日饭卡记账\%Today%_mail.txt

sendmail -t < S:\朱俊禹\每日饭卡记账\%Today%_mail.txt
```

[点击此处下载脚本中所需的sendmail程序](http://skofield.me/assets/dossendmail/sendmail.zip)

<p>使用sendmail程序方法：1.解压到本地磁盘。2.编辑解压目录中的sendmail.ini文件，配置上自己的邮件服务器、用户名和密码。3.在操作系统的环境变量中加上sendmail.exe所在路径</p>
