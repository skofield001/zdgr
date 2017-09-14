---
layout: post
category: "python"
title:  "BL模型的python实现"
---
<br/>

## BL程序文件清单

```
blcode目录内包含如下文件：
blfuncs.py    里面定义了取观点矩阵、BL计算函数，用于在主程序中调用
读原始参数生成日涨跌幅和观点参数表格.py   读取BL初始化参数，生成资产日涨跌幅数据、观点参数表格供输入观点
读观点参数表格进行BL计算.py     读取资产日收益率数据和观点矩阵，计算BL权重
bl_ini.xls    用于输入初始化参数，如资产代码列表、历史数据采样开始时间和结束时间、交易起止日期等
空目录output  用于生成BL权重配置结果和组合在BL模型下回测的日净值
```
<br/>
<!-- more -->
## 使用方法
<p>1.将整个blcode目录拷贝到本地磁盘</p>
<p>2.进入blcode目录，修改BL初始化文件bl_ini.xls，填入资产代码列表、初始权重、风险厌恶系数、日涨跌幅取样开始日期、日涨跌幅取样结束日期、交易开始日期、交易结束日期、历史数据回溯天数。</p>
<font color="#ff0000" face="黑体">填完之后注意保存并退出。</font>
![](http://skofield.me/assets/blrealbypython/blug1.png)
<font color="#ff0000" face="楷体">注：①日期填写以YYYY-MM-DD格式输入对应的excel单元格中</font>
<font color="#ff0000" face="楷体">②日涨跌幅取样开始日期和结束日期为生成资产日收益率表格所用，可取较长日期如以2010-1-1开始，2017-9-1结束，结束日期应覆盖到交易最后一日。</font>

<p>3.启动spyder，打开并运行第一个程序“读原始参数生成日涨跌幅和观点参数表格.py”，程序运行完成后，blcode目录内会出现两个excel文件：bl_view.xls和 资产日收益率.xls</p>
![](http://skofield.me/assets/blrealbypython/blug2.png)

<p>4.打开bl_view.xls，每月都有一个sheet，在对应月份的sheet中填入当月的主观观点。注意：第一个sheet中的第二行第一列必须填1，否则后续无法计算。后续月份仅在当月有新观点时填1并在相应单元格内输入观点矩阵。填完之后注意保存并退出。</p>
![](http://skofield.me/assets/blrealbypython/blug3.png)


<p>5.用spyder打开并运行第二个程序“读观点参数表格进行BL计算.py”，运行完成后，blcode目录下的output目录内会生成bl_result.xls和port_netval.csv，分别记录每月的BL权重和每日组合净值(交易起始日净值为1)。</p>
![](http://skofield.me/assets/blrealbypython/blug4.png)

<br/>
## 代码文件内容：

### blfuncs.py
```
# -*- coding: utf-8 -*-
"""
Created on Fri Jun 23 15:26:52 2017

@author: skofield
"""

import pandas as pd
import numpy as np
import xlrd
from cvxopt import solvers,matrix

#定义是否为debug模式，该模式下函数会输出一些中间变量供调试
debug_mode=0


#***********************************************************************
#                                                                      *
# 定义根据取sheet中新观点P,Q及信心水平LC的函数get_pqc,参数data为               *
# 读取的excel文件数据，view_sheet为sheet号,stock_count为股票个数             *
#                                                                      *
#***********************************************************************
def getpqc(bl_view_filepath,view_sheet,stock_count):
  bl_view_data=xlrd.open_workbook(bl_view_filepath)
  cftable=bl_view_data.sheets()[view_sheet]
  view_count=0
  for j in range(7,cftable.nrows):
    if(cftable.cell(j,1).value!=''):
      view_count+=1
    else:
      break
   
    #读取观点矩阵P
  P=list(range(view_count))
  for j in range(0,view_count):
    P[j]=cftable.row_values(7+j)[1:stock_count+1]
  P=np.mat(P)

  #读取观点信心水平LC
  LC=[]
  for i in range(0,view_count):
    LC.append(cftable.cell(7+i,stock_count+1).value)

   #读取观点超额收益矩阵Q
  Q=[]
  for j in range(0,view_count):
    Q.append(cftable.cell(20+j,1).value)
  Q=np.mat(Q).T
          
  #函数返回矩阵P,Q,观点信心列表LC,观点个数view_count
  return P,Q,LC,view_count
#******************************getpqc函数定义结束*******************************




#***********************************************************************
#                                                                      *
#                     定义BL计算函数                                    *
#                                                                      *
#***********************************************************************

def bl(daily_r_path,his_start_date,his_end_date,delta,w_mkt,P,Q,LC,view_count):
  
  #打开日收益率表格的第一个sheet,即日收益率表格
  daily_r_data=xlrd.open_workbook(daily_r_path)
  daily_r_table=daily_r_data.sheets()[0]
  
  #取日收益率表格第一列的时间序列并化为datetime格式
  date=daily_r_table.col_values(0)[3:]
  date=pd.to_datetime(date)
  
  #取股票日收益率数据，转换成Series
  stock_count=daily_r_table.ncols-1
  stock_r=list(range(stock_count))
  for i in range(0,stock_count):
    stock_r[i]=daily_r_table.col_values(i+1)[3:]
    stock_r[i]=pd.Series(stock_r[i],index=date)
    stock_r[i]=stock_r[i][his_start_date:his_end_date]
    for j in range(0,len(stock_r[0])):
      stock_r[i][j]*=0.01
  #定义股票日收益率的年化协方差矩阵epsi
  epsi=[]
  for i in range(0,stock_count):
    epsi.append(stock_r[i])
  epsi=np.cov(epsi,ddof=1)*250
  epsi=np.mat(epsi)
  
  if debug_mode==1:
    print("epsi:")
    print(epsi)
    print("LC:")
    print(LC)
  



######################BL模型计算过程开始######################################
  



#&&&&&&&&&取历史数据终止日期的市值权重来计算隐含均衡收益率矩阵pai&&&&&

  #计算隐含均衡收益率矩阵pai
  pai=delta*epsi*w_mkt    
  

  if(debug_mode==1):
    print("pai:")
    print(pai)
  #pai=np.mat([0.22707,0.21833,0.19397,0.2034,0.15009,0.17566,0.16312,0.21116,0.17238])
  #pai=pai.T  
  #&&&&&&&&&&&&&&&&计算BL模型下的预期收益率矩阵E_bl&&&&&&&&&&&&&&&&&&
  #计算P_star矩阵,P_star矩阵为P按列求和生成的1*k矩阵，k表示观点数
  P_star=sum(P)
  
  if debug_mode==1:
    print('P_star:')
    print(P_star)
  
  #中间变量pep，用于计算标准刻度因子
  pep=P_star*epsi*P_star.T
  
  if debug_mode==1:
    print("pep:")
    print(pep)
  
  
  #标准刻度因子CF
  CF=float(0.5*pep)

  if debug_mode==1:
    print("CF:")
    print(CF)  
  
  

  #计算看法置信度矩阵omega
  cfli=0
  omega=[]
  for i in range(0,view_count):
    cfli+=CF/LC[i]
    omega.append(CF/LC[i])

  #计算刻度因子tao
  tao=float(pep*view_count/cfli)

  omega=np.diag(omega)
  omega=np.mat(omega)
  #omega=np.diag(P*(tao*epsi)*P.T)
  #omega=np.mat(omega)
  
  if debug_mode==1:
    
    print("omega:")
    print(omega)
  
  


  
  #计算新的加权后的收益向量E_bl
  E_bl=((tao*epsi).I+P.T*omega.I*P).I*((tao*epsi).I*pai+P.T*omega.I*Q)
  #E_bl=pai+tao*epsi*P.T*(omega+tao*P*epsi*P.T).I*(Q-P*pai)
    
  #E_bl=np.linalg.inv(te_ni+P.T*omega_ni*P)*(te_ni*pai+P.T*omega_ni*Q)
  
  #E_bl=np.mat([0.25784,0.25183,0.21006,0.22032,0.16402,0.19481,0.15941,0.23495,0.18543])
  #E_bl=E_bl.T
  
  #&&&&&&&&&&&&&&&&&&&&&&计算新的最优组合权重&&&&&&&&&&&&&&&&&&&&&&&&
  #构造cvxopt公式中的参数矩阵p,q
  p=matrix(delta*epsi)
  q=matrix(-1*E_bl)

  #构造参数矩阵A,b
  A=[]
  for i in range(0,stock_count):
    A.append(1.0)
  A=np.mat(A)
  A=matrix(A)

  b=np.mat([1.0])
  b=matrix(b)
    
  #构造参数矩阵G
  G=[]
  for i in range(0,stock_count):
    G.append(-1.0)
  G=np.diag(G)
  G=matrix(G)


  #构造参数矩阵h
  h=[]
  for i in range(0,stock_count):
    h.append(0.0)
  h=matrix(h)
  
  #计算BL模型下最优权重矩阵w_bl
  sol=solvers.qp(p,q,G,h,A,b)
  w_bl=sol['x']
  w_bl=np.mat(w_bl)
  
  if debug_mode==1:
    print("delta="+str(delta))
    print("tao="+str(tao))
    #print("pai="+str(pai))
    print("E_bl:")
    print(E_bl)
  
  #返回市值权重矩阵w_mkt和BL模型下最优权重矩阵w_bl
  return w_bl    
  
  #*****************************bl函数定义结束******************************

```

### 读原始参数生成日涨跌幅和观点参数表格.py
```
# -*- coding: utf-8 -*-
"""
Created on Fri Sep  1 16:01:08 2017

@author: skofield
"""
import os,sys
import xlrd
import xlwt
from xlutils.copy import copy
from WindPy import w
import datetime
import pandas as pd
import time
from xlrd import xldate_as_tuple

#取当前目录作为工作目录,该目录用于程序运行所有文件的生成
working_dir=os.getcwd()

print("当前工作目录为："+working_dir)  

#定义BL初始化参数文件路径
ini_file_path=working_dir+"\\" + "bl_ini.xls"


#打开BL初始化参数文件，读取第一个sheet
ini_data=xlrd.open_workbook(ini_file_path)
ini_table=ini_data.sheets()[0]

#读取资产代码列表stock_list和资产名称stock_name
stock_list=ini_table.row_values(0)[1:]
stock_name=ini_table.row_values(1)[1:]


#读取历史数据开始日期和结束日期并格式化处理，用于生成历史收益率表格
his_start_date=ini_table.cell(7,1).value
his_end_date=ini_table.cell(9,1).value
his_start_date=datetime.datetime(*xldate_as_tuple(his_start_date,0)).date()
his_end_date=datetime.datetime(*xldate_as_tuple(his_end_date,0)).date()

#读取交易开始日期和结束日期并格式化处理，用于生成BL观点表格
trade_start_date=ini_table.cell(11,1).value
trade_end_date=ini_table.cell(13,1).value
trade_start_date=datetime.datetime(*xldate_as_tuple(trade_start_date,0)).date()
trade_end_date=datetime.datetime(*xldate_as_tuple(trade_end_date,0)).date()



#*****************************************************************************
#                                                                            *
#                           生成资产日收益率表格                                 *
#                                                                            * 
#*****************************************************************************

#定义资产日收益率表格生成路径
daily_r_path=working_dir+"\\"+"资产日收益率.xls"

#从wind读取数据，生成日收益率数据表格
w.start()

stock_name=[]
for i in range(0,len(stock_list)):
  stock_name.append(w.wsd(stock_list[i],'sec_name').Data[0][0])



his_date=w.tdays(his_start_date,his_end_date).Data[0]
for i in range(0,len(his_date)):
  his_date[i]=his_date[i].date()
his_date=pd.to_datetime(his_date)


             
stock_r=list(range(len(stock_list)))
for i in range(0,len(stock_list)):
  stock_r[i]=w.wsd(stock_list[i],'pct_chg',his_start_date,his_end_date,\
  'PriceAdj=F').Data[0]
  stock_r[i]=pd.Series(stock_r[i],index=his_date)

#创建资产日收益率工作簿
assets=xlwt.Workbook() 

#创建第一个sheet：
sheet1=assets.add_sheet(u'资产日收益率',cell_overwrite_ok=True)
#生成第一行
for i in range(0,len(stock_list)):
  sheet1.write(0,i+1,stock_list[i])
  sheet1.write(1,i+1,stock_name[i])
  sheet1.col(i+1).width = (len('沪深300工业')*460)

for i in range(0,len(stock_list)):
  sheet1.write(2,i+1,'涨跌幅(%)')


for i in range(0,len(his_date)):
  sheet1.write(i+3,0,his_date[i].strftime("%Y-%m-%d"))
sheet1.col(0).width = (len('yyyy-mm-dd')*300)

style1=xlwt.XFStyle()
fmt='##0.0000'
style1.num_format_str=fmt
for i in range(0,len(stock_list)):
  for j in range(0,len(his_date)):
    sheet1.write(j+3,i+1,stock_r[i][j],style1)

assets.save(daily_r_path)

print("资产日收益率表格生成完毕！表格路径为："+daily_r_path)




#*****************************************************************************
#                                                                            *
#                           生成观点参数表格                                    *
#                                                                            * 
#*****************************************************************************

#定义BL观点参数表格生成路径
bl_view_file_path=working_dir+"\\"+"bl_view.xls"

#创建BL观点参数表格
bl_view=xlwt.Workbook()

#取模拟交易区间的所有月份
begin_date=trade_start_date
end_date=trade_end_date
trade_month_list=[]
while begin_date <= end_date:
  date_str=begin_date.strftime("%Y%m")
  trade_month_list.append(date_str)
  begin_date+=datetime.timedelta(days=1)
trade_month_list=list(set(trade_month_list))
trade_month_list.sort()


#对模板文件进行操作，每个交易月份增加一个sheet，用于输入相关信息
for i in trade_month_list:
  ws=bl_view.add_sheet(i+"观点",cell_overwrite_ok=True)
  temp_str=i+'是否有新观点:(如有则在第二行第一列单元格填1,并在下方输入新的P、Q)'
  ws.write(0,0,temp_str)
  ws.write(3,1,'新的观点矩阵P：')
  ws.write(7,0,'观点1:')
  ws.write(8,0,'观点2:')
  ws.write(9,0,'...')
  ws.write(19,1,'新的观点超额收益矩阵Q：')
  ws.write(20,0,'观点1超额收益：')
  ws.write(21,0,'观点2超额收益：')
  ws.write(22,0,'...')
  ws.col(0).width = (len('观点2超额收益：')*460)
  for j in range(0,len(stock_list)):
    ws.write(5,j+1,stock_list[j])
    ws.col(j+1).width = (len('沪深300工业')*460)
    ws.write(6,j+1,stock_name[j])
  ws.write(6,len(stock_list)+1,'信心水平')
bl_view.save(bl_view_file_path)

print("BL参数表格生成完毕，表格路径为："+bl_view_file_path+",请打开表格填写相关参数！")

```

### 读观点参数表格进行BL计算.py
```
# -*- coding: utf-8 -*-
"""
Created on Mon Sep  4 16:04:48 2017

@author: skofield
"""

import os
from bl_funcs import getpqc
from bl_funcs import bl
import xlrd
import numpy as np
import pandas as pd
from WindPy import w
import datetime
from xlrd import xldate_as_tuple
import xlwt

#读取当前目录作为工作目录
working_dir=os.getcwd()

#定义BL初始化参数文件路径
bl_ini_filepath=working_dir+"\\"+"bl_ini.xls"

#定义资产日收益率表格路径
daily_r_path=working_dir+"\\"+"资产日收益率.xls"

#定义BL观点参数表格路径
bl_view_filepath=working_dir+"\\"+"bl_view.xls"

#打开BL初始化参数文件、资产日收益率文件、BL观点参数文件
bl_ini_data=xlrd.open_workbook(bl_ini_filepath)
daily_r_data=xlrd.open_workbook(daily_r_path)
bl_view_data=xlrd.open_workbook(bl_view_filepath)

#读取BL初始化参数文件中的交易开始日期和结束日期并格式化处理
bl_ini_table=bl_ini_data.sheets()[0]
trade_start_date=bl_ini_table.cell(11,1).value
trade_end_date=bl_ini_table.cell(13,1).value
trade_start_date=datetime.datetime(*xldate_as_tuple(trade_start_date,0)).date()
trade_end_date=datetime.datetime(*xldate_as_tuple(trade_end_date,0)).date()
trade_start_date=str(trade_start_date)
trade_end_date=str(trade_end_date)

#取BL初始化参数文件中的初始市场权重
w_mkt=bl_ini_table.row_values(3)[1:]
w_mkt=np.mat(w_mkt).T

#取BL初始化参数文件中的股票代码列表并计数
stock_list=bl_ini_table.row_values(0)[1:]
stock_count=len(stock_list)

#读取资产名称列表stock_name
stock_name=bl_ini_table.row_values(1)[1:]
            
#读取风险厌恶系数delta
delta=bl_ini_table.cell(5,1).value

#读取回溯天数
recall_days=bl_ini_table.cell(15,1).value                       
                       
#读取资产日收益率表格数据构造dataframe
daily_r_df=pd.read_excel(daily_r_path,sheetname=0,header=0,index_col=0)
daily_r_df.index=pd.to_datetime(daily_r_df.index)

#取交易区间内的交易日，构造交易时间序列
trade_series=daily_r_df[trade_start_date:trade_end_date].index


###############################################################################
#                                                                             #
#                             交易开始                                          #
#                                                                             #
###############################################################################

#在交易区间内，按月调仓进行操作。每进入下个月，取最近一年历史统计数据，按需读取新观点
view_sheet=-1
cur_month=''

#定义BL组合净值list
port_netval=[]

#新建一个excel，用于记录每月BL计算出的最新权重配置
monthly_bl_w=xlwt.Workbook()
#定义上面的excel存放路径
bl_result_filepath=working_dir+'\\output\\bl_result.xls'

#对交易日期进行遍历循环
for i in range(0,len(trade_series)):
  #判断是否进入下个月
  if(str(trade_series[i])[0:7]!=cur_month):
    #进入新的月份后，按照回溯天数重新回溯数据
    recall_start_date=trade_series[i]-datetime.timedelta(recall_days)
    recall_end_date=trade_series[i]-datetime.timedelta(1)
    #当前月份重新赋值
    cur_month=str(trade_series[i])[0:7]
    
    #取下一个sheet
    view_sheet+=1    

    #判断本月对应的sheet是否有新观点，如有，则读取新观点
    if bl_view_data.sheets()[view_sheet].cell(1,0).value==1:
      P,Q,LC,view_count=getpqc(bl_view_filepath,view_sheet,stock_count)

    #计算本月BL权重
    w_bl=bl(daily_r_path,recall_start_date,recall_end_date,delta,w_mkt,P,Q,LC,view_count)
    
    
    #在excel中新建一个sheet，记录本月计算出的权重结果
    new_sheet=monthly_bl_w.add_sheet(cur_month,cell_overwrite_ok=True)
    
    #定义BL结果写入excel时显示保留四位小数
    style1=xlwt.XFStyle()
    fmt='##0.0000'
    style1.num_format_str=fmt
    new_sheet.write(0,0,'资产列表: ')
    new_sheet.write(3,0,'当月BL权重: ')
    new_sheet.col(0).width = (len('当月BL权重')*460)
    for j in range(0,len(stock_list)):
      new_sheet.write(0,j+1,stock_list[j])
      new_sheet.write(1,j+1,stock_name[j])
      new_sheet.write(3,j+1,float(w_bl[j]),style1)
      new_sheet.col(j+1).width = (len('沪深300工业')*460) #设置excel列宽
    

    
  
  #计算每日组合净值
  net_val=float(np.mat(daily_r_df[str(trade_series[i])]*0.01+1)*w_bl)
  port_netval.append(net_val)

#将每月的BL权重配置保存到文件
monthly_bl_w.save(bl_result_filepath)

#对BL组合日净值list加上时间序列索引，构造为Series
port_netval=pd.Series(port_netval,index=trade_series,name='BL组合日净值')
  
#定义每日组合净值保存到csv文件的路径，将每日净值存入
port_netval_filepath=working_dir+'\\output\\port_netval.csv'
port_netval.to_csv(port_netval_filepath)

```
