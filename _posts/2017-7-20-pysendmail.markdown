---
layout: post
category: "python"
title:  "python读取excel生成邮件并发送"
---
<br/>

<p>近日，实习指导老师要求我将产品日收益率表格数据生成邮件并自动发送给客户，日收益率表格数据类似如下这种：</p>
![](http://skofield.me/assets/pysendmail/exl_sample.jpg)  

<!-- more -->

<p>单纯的将excel文件放在邮件附件中还需要客户下载附件，既不直观也给客户造成了一定的不便。我上网查了下，发现邮件还可以以html方式发送，即邮件服务器可以识别html语言表达出的丰富内容并展示给收件人。我们经常受到的排版精良并含有各种精美动态内容的广告(laji)邮件即是通过此种方式发出的。学习了一些html的基础语法之后，决定采用table方式，用python抓取excel文本内容，并自动生成html代码，测试之后可以成功发出，达到了指导老师要求，客户受到的邮件效果如下：</p>
![](http://skofield.me/assets/pysendmail/mail_sample.jpg)

<p>为实现这一功能，写了两个代码，html_mail_gen.py和 python发邮件测试.py，第一个代码文件包含一个将excel表格内容转成html表格的函数html_mail_gen( )，第二个代码文件调用该函数生成邮件正文，然后通过python邮件包发出该邮件。两个代码文件内容如下：</p>

##### html_mail_gen.py
```
import pandas as pd

def html_mail_gen(df,title,filepath):

  #df=pd.read_csv('d:/concat.csv',encoding='gbk',index_col=0)

  columns=list(df.columns)
  index=list(df.index)
      
  f=open(filepath,'w')
  f.write('<div style="line-height:1.7;color:#000000;font-size:14px;font-family:Arial">\n\n')
  f.write("<style>\n")
  f.write("table,table tr th, table tr td { border:1px solid #0094ff; }\n")
  f.write("table { width: auto; min-height: 25px; line-height: 25px; \
   text-align: center; border-collapse: collapse; padding:2px;} \n")
  f.write("</style>\n\n")
  f.write("<table>\n")
  f.write("<caption>"+title+"</caption>\n")
  f.write("<tbody>\n")
  
  colstr="<tr height=17 style='height:12.75pt'>\n"
  colstr+="<td></td>\n"
  for i in range(0,len(columns)):
    colstr+='<td>'+columns[i]+'</td>\n'
  colstr+='</tr>'
  f.write(colstr+'\n')
  f.write('\n')
  
  for i in range(0,len(index)):
    rowstr="<tr height=17 style='height:12.75pt'>\n"+"<td>"+str(index[i])+"</td>\n"
    for j in range(0,len(columns)):
      rowstr+="<td>"+str(df.iat[i,j])+"</td>\n"
    rowstr+="</tr>\n"
    f.write(rowstr)
      
  f.write('</tbody>\n</table>\n</div>')
  f.close()
```

<br/>

#### python发邮件测试.py
```
import smtplib
from email.mime.text import MIMEText
from email.header import Header
from html_mail_gen import html_mail_gen

filepath1='d:/concat1.csv'
filepath2='d:/concat2.csv'
filepath3='d:/concat3.csv'

df1=pd.read_csv(filepath1,encoding='gbk',index_col=0)
df2=pd.read_csv(filepath2,encoding='gbk',index_col=0)
df3=pd.read_csv(filepath3,encoding='gbk',index_col=0)

title1='资产日收益率1'
title2='资产日收益率2'
title3='资产日收益率3'

mail_content1=html_mail_gen(df1,title1)
mail_content2=html_mail_gen(df2,title2)
mail_content3=html_mail_gen(df3,title3)


content=mail_content1+mail_content2+mail_content3
print(content)


# 第三方 SMTP 服务
mail_host = "smtp.163.com"  # SMTP服务器
mail_user = "zhudgr"  # 用户名
mail_pass = "*******"  # 密码


sender = 'zhudgr@163.com'  # 发件人邮箱
receivers = ['skofield@163.com','zhujunyu@cpic.com.cn']  
title='PYTHON SMTP MAIL TEST'

message = MIMEText(content, 'html', 'utf-8')
message['From'] = "{}".format(sender)
message['To'] =   ",".join(receivers)

message['Subject'] = title 

try:
    smtpObj = smtplib.SMTP(mail_host, 25)  
    smtpObj.login(mail_user, mail_pass)  
    smtpObj.sendmail(sender, receivers, message.as_string())  
    print("mail has been send successfully.")
except smtplib.SMTPException as e:
    print(e)
```

<font color="#ff0000" face="黑体">注意：使用python发送邮件需要在邮箱中设置启用smtp服务并设置密码，此处不展开介绍。</font>
