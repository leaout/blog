---
title: bcolz-股票行情数据压缩神器
date: 2024-01-05 22:26:08
tags:
---
# bcolz carray结合股票数据优化压缩使用

官方文档：https://pypi.org/project/bcolz/

​
之所以使用bcolz，看中了他的高压缩率，数字类型数据压缩率很高。

比如说，证券行业的历史数据，数据内容包括时间，价格等信息，都是数字类型。对比csv文本文件来说，可以压缩到之前的1/4.

举个栗子:

这是某一证券的分钟历史k线文本，占用空间45M

![1](/images/bcolz-1.jpg)


文件格式类型如下，都是数字 

![2](/images/bcolz-2.jpg)

接下来看使用carray 存储的结果 11.4M 

![3](/images/bcolz-3.jpg)



最后献上代码
```python
#arr 为numply 数组
carr = bcolz.carray(arr, chunklen=10 * 1024,expectedlen=10*1024,rootdir="D:/quote/000001.XSHE1",cparams=bcolz.cparams(quantize=1))
#存磁盘
carr.flush() 
#格式转换过程：csv->numpy->bcolz.carray
```


函数中chunklen 为存储块长度，类似数据文件的页，expectedlen 预期长度，设置之后方便更快速的存取数据，

quantize 是压缩因子，官方提供了1，2，3三种压缩，经测试后，quantize=1压缩率最高。

附上例子 https://github.com/leaout/curs/blob/master/curs/data_source/buddle_tool.py

​