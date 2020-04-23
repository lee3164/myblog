# wal日志

## 什么是wal日志？
WAL: Write-Ahead Logging 预写日志系统

数据库中一种高效的日志算法，对于非内存数据库而言，磁盘I/O操作是数据库效率的一大瓶颈。在相同的数据量下，采用WAL日志的数据库系统在事务提交时，磁
盘写操作只有传统的回滚日志的一半左右，大大提高了数据库磁盘I/O操作的效率，从而提高了数据库的性能。

## prometheus的wal
prometheus的tsdb在数据commit的时候先写wal日志，如果出现问题则rollback。

### 数据格式

|内容|类型|占用空间|备注|
| --- | ---| --- | --- |
|Series的类型|byte|1字节||
|ref|i64|8字节||
|labels的数量|uvarint|-||
|label key|uvarintStr|-||
|label value|uvarintStr0|-||
|-|-|-|...重复label的key和value||
|Samples的类型|byte|1字节||
|samples[0].ref|i64|8字节||
|samples[0].T|i64|8字节||
|-|-|-|从现在开始重新遍历samples，ref，t存的都是与samples[0]的对应值的差值|
|ref delta|varint64|-|-|
|T delta|varint64|-|-|
|v|float64|8字节||

存差值的好处在于更节省空间，因为对于tsdb来说，t和v的变化都是相对固定，大部分情况下t，v的差值变化很小，甚至可能都是0。缺点就是要得到所有值需要迭代
计算。

### 存储格式

上面我们说到了数据格式，存储格式又是什么意思？针对上层来说，存储层是一个抽象概念，可以做到无感知。存储层存在的意义则是最大化的利用磁盘使用率，
利用各种磁盘特性减小内存，磁盘存在的速度差异。先看以下几个概念：

- Segment segment代表的是一个文件，一个segment是128M。
- Page 代表一次性从磁盘读取或写入的大小，一个page是32K

整体设计和leveldb很类似，如果一个page能够存下整个数据，就存在一个page内，知道一个page存不下为止。根据page可用大小可能会将数据分成多段，有以下
4种类型：

- Full 代表数据可以完整存在当前的page
- First 代表数据的第一段存在当前page
- Middle 代表数据的中间部分存在这个page
- Last 代表数据的最后一段存在这个page

数据的元信息存储在数据的开头的7个字节，分别是数据类型，就是上面的4个类型(uint8)。然后是数据长度（uint16），最后是CRC32（int32）用于校验数据
完整性。





