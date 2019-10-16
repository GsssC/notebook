# Mapreduce原理和YARN 
||**传统关系型数据库**|**MapReduce**|
|-|-|-|
|**数据大小**|GB|PB|
|**访问**|交互型和批处理|批处理|
|**更新**|多次读写|一次写入多次读取|
|**结构**|静态模式|动态模式|
|**集成度**|高|低|
|**伸缩性**|非线性|线性|
## MapReduce特性
- 自动实现分布式并行计算
- 容错
- 提供状态监控工具
- 模型抽象

Map()输入是表示一行的一对键值对，`<行首字母的字符偏移量，文本>`
```
hello sb
hello fool
hello nerd
```
上述三行可以分解为`<0,'hello sb'>`,`<9，'hello fool'>`,`20,'hello nerd'`
## MapReduce执行流程-MRv1(老版本)
- 开始
- 向jobTracker申请JobID
- 配置Job运行环境
- 检查Job输出配置
- 切分Job的输入数据
- 生成Job的Job.xml
- 向JobTracker提交Job.xml
## MapReduce执行流程-MRv2

