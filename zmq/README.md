# DolphinDB zmq Plugin

## 1. 安装构建

### 1.1 预编译安装

#### Linux

预先编译的插件文件存放在DolphinDBPlugin/zmq/bin/linux64目录下。通过DolphinDB，执行以下命令可导入zmq插件：

```
cd DolphinDB/server //进入DolphinDB server目录
./dolphindb //启动 DolphinDB server
loadPlugin("<PluginDir>/zmq/bin/linux64/PluginZmq.txt") //加载插件
```

### 1.2 自行编译

#### 编译libzmq

下载 [libzmq-4.3.4]https://github.com/zeromq/libzmq/releases/tag/v4.3.4
```bash
cd libzmq-4.3.4
cp include/zmq.h /path/to/PluginZmq/bin/include/
mkdir build && cd build
cmake ..
make -j8
cp lib/libzmq.a /path/to/PluginZmq/bin/linux64/
```

#### 获取cppzmq头文件

下载 [cppzmq-4.7.1]https://github.com/zeromq/cppzmq/releases/tag/v4.7.1
```bash
cd cppzmq-4.7.1
cp zmq.hpp /path/to/PluginZmq/bin/include/
```

#### 构建插件：

构建插件内容：

```
mkdir build
cd build
cmake  ../
make
```

**注意**：编译之前请确保libDolphinDB.so在gcc可搜索的路径中。可使用`LD_LIBRARY_PATH`指定其路径，或者直接将其拷贝到build目录下。

编译后目录下会产生两个文件：libPluginZmq.so和PluginZmq.txt。

##  2. 发送

### 2.1 zmq::socket

#### 语法

zmq::socket(type, formatter, [batchSize], [prefix])

#### 详情

创建一个zmq socket。

**注意**：对connect, bind, send, close接口进行并发操作时，需要为各线程创建不同的zmq socket连接句柄。

#### 参数

* type: 为STRING类型，表示要创建的socket类型，取值为"ZMQ_PUB"和"ZMQ_PUSH"。
* formatter: 一个函数，用于指定发布的数据的打包格式，包括CSV或JSON两种格式。由`createJSONFormatter`或`createCSVFormatter`创建。
* batchSize: 为INT类型，表示每次发送的记录行数。当待发布内容是一个表时，可以进行分批发送。
* prefix: 为STRING类型，表示发送前缀。

#### 例子
```
formatter = zmq::createJSONFormatter()
socket = zmq::socket("ZMQ_PUB", formatter)
```

### 2.2 zmq::connect

#### 语法

zmq::connect(socket, addr, [prefix])

#### 详情

进行zmq的socket连接。

#### 参数
* socket: zmq连接句柄。
* addr: 是一个STRING，格式为"transport://address:port"，表示socket连接到的远端地址。transport表示要使用的底层协议，取值为tcp, ipc, inproc, pgm或epgm。address:port表示远端的IP地址和端口号。
* prefix: 发送前缀，类型为STRING。

#### 例子
```
formatter = zmq::createJSONFormatter()
socket = zmq::socket("ZMQ_PUB", formatter)
zmq::connect(socket, "tcp://localhost:55632", "prefix1")
```

### 2.3 zmq::bind

#### 语法

zmq::bind(socket, addr, [prefix])

#### 详情

绑定socket，接收发来的链接请求。

#### 参数
* socket: zmq连接句柄。
* addr: 是一个STRING，格式为"transport://address:port"，表示socket绑定的地址。transport表示要使用的底层协议，取值为tcp, ipc, inproc, pgm或epgm。address:port表示进行绑定的地址和端口号，*表示同一个服务器的所有IP。
* prefix: 发送前缀，类型为STRING。

#### 例子
```
formatter = zmq::createJSONFormatter()
socket = zmq::socket("ZMQ_PUB", formatter)
zmq::bind(socket, "tcp://*:55631", "prefix1")
```

### 2.4 zmq::send

#### 语法

zmq::send(socket, data, [prefix])

#### 详情

发送一条zmq消息。

#### 参数
* socket: zmq连接句柄。
* data: 发送的数据，为一个表。
* prefix: 消息前缀，为STRING类型。

#### 返回值

如果发送成功，返回true。

#### 例子
```
formatter = zmq::createJSONFormatter()
socket = zmq::socket("ZMQ_PUB", formatter)
zmq::connect(socket, "tcp://localhost:55632", "prefix1")
zmq::send(socket, table(1..10 as id))
```

### 2.5 zmq::close

#### 语法

zmq::close(socket)

#### 详情

关闭一个zmq连接句柄。

#### 例子
```
formatter = zmq::createJSONFormatter()
socket = zmq::socket("ZMQ_PUB", formatter)
zmq::connect(socket, "tcp://localhost:55632", "prefix1")
zmq::close(socket)
```

## 3. 订阅

### 3.1 zmq::createSubJob

#### 语法

zmq::createSubJob(addr, type, isConnnect, handle, parser, [prefix])

#### 详情

创建一个zmq订阅。

#### 参数

* addr: 是一个STRING，格式为"transport://address:port"。transport表示要使用的底层协议，取值为tcp, ipc, inproc, pgm或epgm。address:port表示zmq绑定的地址和端口。
* type: 为STRING类型，表示zmq的socket类型，取值为"ZMQ_SUB"h和"ZMQ_PULL"。
* isConnnect: 为BOOL类型，是否是对addr进行连接，如果为否，则对addr进行绑定，。
* handle: 一个函数或表，用于处理从zmq接收的消息。
* parser: 一个函数，用于对发布的数据按CSV或JSON格式进行解包。由`createJSONParser`或`createCSVParser`创建。
* prefix: 为STRING类型，表示消息前缀。

#### 例子
```
handle = streamTable(10:0, [`int], [INT])
enableTableShareAndPersistence(table=handle, tableName=`test1, asynWrite=true, compress=true, cacheSize=10000000, retentionMinutes=120)
parser = zmq::createJSONParser([INT], [`bool])
zmq::createSubJob("tcp://localhost:55633", "ZMQ_SUB", handle, parser, "prefix1")
```

### 3.2 zmq::getSubJobStat

#### 语法

zmq::getSubJobStat()

#### 详情

查询所有zmq订阅信息。

查询所有订阅信息。返回一个表，包含如下字段：
* subscriptionId: 表示订阅标识符。
* addr: zmq订阅地址。
* prefix: zmq订阅前缀。
* recvPackets: zmq订阅订阅收到的消息报文数。
* createTimestamp: 表示订阅建立时间。

#### 例子
```
handle = streamTable(10:0, [`int], [INT])
enableTableShareAndPersistence(table=handle, tableName=`test1, asynWrite=true, compress=true, cacheSize=10000000, retentionMinutes=120)
parser = zmq::createJSONParser([INT], [`bool])
zmq::createSubJob("tcp://localhost:55633", "ZMQ_SUB", handle, parser, "prefix1")
zmq::getSubJobStat()
```


### 3.3 zmq::cancelSubJob

#### 语法

zmq::cancelSubJob(subscription)

#### 详情

关闭一个zmq订阅。

#### 参数

* subscription: 是`createSubJob`函数返回的值或`getJobStat`返回的订阅标识符。

#### 例子
```
zmq::cancelSubJob(sub1)
zmq::cancelSubJob(42070480)
```

### 3.4 zmq::zmqCreatepusher

#### 语法

zmq::zmqCreatepusher(socket, dummyTable)

#### 详情

创建一个zmq的pusher，支持用append方法和作为流数据引擎的outputTable。

#### 参数

* socket: zmq的socket。
* dummyTable: 提供输入参数的schema。

#### 例子
```
share streamTable(1000:0, `time`sym`volume, [TIMESTAMP, SYMBOL, INT]) as trades
output1 = table(10000:0, `time`sym`sumVolume, [TIMESTAMP, SYMBOL, INT])

formatter = zmq::createJSONFormatter()
socket = zmq::socket("ZMQ_PUB", formatter)
zmq::connect(socket, "tcp://localhost:55632")
pusher = zmq::createPusher(socket, output1)

engine1 = createTimeSeriesEngine(name="engine1", windowSize=60000, step=60000, metrics=<[sum(volume)]>, dummyTable=trades, outputTable=pusher, timeColumn=`time, useSystemTime=false, keyColumn=`sym, garbageSize=50, useWindowStartTime=false)
subscribeTable(tableName="trades", actionName="engine1", offset=0, handler=append!{engine1}, msgAsTable=true);

insert into trades values(2018.10.08T01:01:01.785,`A,10)
insert into trades values(2018.10.08T01:01:02.125,`B,26)
insert into trades values(2018.10.08T01:01:10.263,`B,14)
insert into trades values(2018.10.08T01:01:12.457,`A,28)
insert into trades values(2018.10.08T01:02:10.789,`A,15)
insert into trades values(2018.10.08T01:02:12.005,`B,9)
insert into trades values(2018.10.08T01:02:30.021,`A,10)
insert into trades values(2018.10.08T01:04:02.236,`A,29)
insert into trades values(2018.10.08T01:04:04.412,`B,32)
insert into trades values(2018.10.08T01:04:05.152,`B,23)
```

## 4. 打/解包功能

### 4.1 createCSVFormatter

```
zmq::createCSVFormatter([format], [delimiter=','], [rowDelimiter=';'])
```
创建一个CSV格式的Formatter函数。

参数：
- 'format': 是一个STRING类型的向量。
- 'delimiter': 是列之间的分隔符，默认是','。
- 'rowDelimiter': 是行之间的分隔符，默认是';'。

例子：
```
MyFormat = take("", 5)
MyFormat[2] = "0.000"
f = createCSVFormatter(MyFormat, ',', ';')
```

### 4.2 createCSVParser

```
zmq::createCSVParser(schema, [delimiter=','], [rowDelimiter=';'])
```
创建一个CSV格式的Parser函数。

参数：
- 'schema': 是一个包含各列数据类型的向量。
- 'delimiter': 是列之间的分隔符，默认是','。
- 'rowDelimiter': 是行之间的分隔符，默认是';'。

例子：
```
def createT(n) {
    return table(take([false, true], n) as bool, take('a'..'z', n) as char, take(short(-5..5), n) as short, take(-5..5, n) as int, take(-5..5, n) as long, take(2001.01.01..2010.01.01, n) as date, take(2001.01M..2010.01M, n) as month, take(time(now()), n) as time, take(minute(now()), n) as minute, take(second(now()), n) as second, take(datetime(now()), n) as datetime, take(now(), n) as timestamp, take(nanotime(now()), n) as nanotime, take(nanotimestamp(now()), n) as nanotimestamp, take(3.1415, n) as float, take(3.1415, n) as double, take(`AAPL`IBM, n) as string, take(`AAPL`IBM, n) as symbol)
}
t = createT(100)
f = zmq::createCSVFormatter([BOOL,CHAR,SHORT,INT,LONG,DATE,MONTH,TIME,MINUTE,SECOND,DATETIME,TIMESTAMP,NANOTIME,NANOTIMESTAMP,FLOAT,DOUBLE,STRING,SYMBOL])
s=f(t)
p = zmq::createCSVParser([BOOL,CHAR,SHORT,INT,LONG,DATE,MONTH,TIME,MINUTE,SECOND,DATETIME,TIMESTAMP,NANOTIME,NANOTIMESTAMP,FLOAT,DOUBLE,STRING,SYMBOL])
p(s)
```

### 4.3 createJSONFormatter

```
zmq::createJSONFormatter()
```
创建一个JSON格式的Formatter函数。

参数：无。

例子：
```
def createT(n) {
    return table(take([false, true], n) as bool, take('a'..'z', n) as char, take(short(-5..5), n) as short, take(-5..5, n) as int, take(-5..5, n) as long, take(2001.01.01..2010.01.01, n) as date, take(2001.01M..2010.01M, n) as month, take(time(now()), n) as time, take(minute(now()), n) as minute, take(second(now()), n) as second, take(datetime(now()), n) as datetime, take(now(), n) as timestamp, take(nanotime(now()), n) as nanotime, take(nanotimestamp(now()), n) as nanotimestamp, take(3.1415, n) as float, take(3.1415, n) as double, take(`AAPL`IBM, n) as string, take(`AAPL`IBM, n) as symbol)
}
t = createT(100)
f = zmq::createJSONFormatter()
f(t)
```
### 4.4 createJSONParser

```
zmq::createJSONParser(schema, colNames)
```
创建一个JSON格式的Parser函数。

参数：
- 'schema' 是一个向量，表示各列的数据类型。
- 'colNames' 是一个向量，表示列名。

例子：
```
def createT(n) {
    return table(take([false, true], n) as bool, take('a'..'z', n) as char, take(short(-5..5), n) as short, take(-5..5, n) as int, take(-5..5, n) as long, take(2001.01.01..2010.01.01, n) as date, take(2001.01M..2010.01M, n) as month, take(time(now()), n) as time, take(minute(now()), n) as minute, take(second(now()), n) as second, take(datetime(now()), n) as datetime, take(now(), n) as timestamp, take(nanotime(now()), n) as nanotime, take(nanotimestamp(now()), n) as nanotimestamp, take(3.1415, n) as float, take(3.1415, n) as double, take(`AAPL`IBM, n) as string, take(`AAPL`IBM, n) as symbol)
}
t = createT(100)
f = zmq::createJSONFormatter()
p = createJSONParser([BOOL,CHAR,SHORT,INT,LONG,DATE,MONTH,TIME,MINUTE,SECOND,DATETIME,TIMESTAMP,NANOTIME,NANOTIMESTAMP,FLOAT,DOUBLE,STRING,SYMBOL],
`bool`char`short`int`long`date`month`time`minute`second`datetime`timestamp`nanotime`nanotimestamp`float`double`string`symbol)
s=f(t)
x=p(s)

```

## 5. 完整例子

```
loadPlugin("/home/zmx/worker/DolphinDBPlugin/zmq/cmake-build-debug/PluginZmq.txt")
go
formatter = zmq::createJSONFormatter()
socket = zmq::socket("ZMQ_PUB", formatter)
zmq::connect(socket, "tcp://localhost:55632")
data = table(1..10 as id, take(now(), 10) as ts, rand(10, 10) as volume)
zmq::send(socket, data)
```
