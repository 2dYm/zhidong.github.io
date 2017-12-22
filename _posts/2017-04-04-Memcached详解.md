---
layout: singlepost
url: 2017-04-04-Memcached详解.md
title: Memcached详解
category: Other
description:  Memcached内存数据库(一个高性能,分布式内存对象缓存系统)
comments: true
---

### Memcached内存数据库(一个高性能,分布式内存对象缓存系统)
<br>

#### 1. 基本介绍 
1.1 参数说明
<img src="http://p07ywvfks.bkt.clouddn.com/Mem-h-e.png" class="img-responsive img-rounded" />
```
-p   设置TCP端口号(默认设置为: 11211)
-d   以daemon方式运行
-u   指定运行Memcache的用户
-l   监听的服务器IP地址，默认应该是本机
-c   设置并发数
-m   允许最大内存用量，单位M (默认: 64 MB)
-v   输出警告和错误信息
-vv  打印客户端的请求和返回信息
-h   打印帮助信息
-i   打印Memcached和libevent的版权信息
```

1.2 启动与停止
```
Memcached的启动:
Memcached -d -p 11211 -u nobody -c 1024 -m 64 

Memcached的关闭:
ps -ef|grep memcached
sudo kill -9 pid(进程号)
```

1.3 连接与退出
```
telnet 127.0.0.1 11211
quit
```
<br>

#### 2. Memcached 命令执行最简单的操作

2.1参数说明
```
command <key> <flags> <expiration time> <bytes>
<value>

command              set/add/replace
key                  key 用于查找缓存值
flags                可以包括键值对的整型参数，客户机使用它存储关于键值对的额外信息
expiration time      在缓存中保存键值对的时间长度（以秒为单位，0表示永远）
bytes                在缓存中存储的字节数
value                存储的值（始终位于第二行）

key是缓存名,
memcached的一个重要特点就是key-value缓存,
即键值对缓存.
每个缓存有一个独特的名字和存储空间.
key是操作数据的唯一标识
问:key能取多长
答:key可以250个字节以内,(不能有空格和控制字符)
注:在新版开发计划中提到key可能会扩充到65535个字节

flag是"标志"的意思,可以用此参数来标志内容的类型.
场景案例:
memcached存储的数据形式只能是字符串.
那么如果要存储 'hello' 和  array('hello','world'); 怎么办?
对于字符串,直接存5个字符即可, 对于array,则需要序列化.
问:取出数据时,又如何处理呢?
字符串,取回直接用, 数组,则需要反序列化成数组.
如何知道,取出的是一段"裸字符串",还是"数组序列化后的字符串"?
答:flag!
标志flag的范围
0-2^16-1
```

2.2 五种基本 memcached 命令
```
增: add
注: 仅当缓存中不存在键时，add 命令才会向缓存中添加一个键值对。如果缓存中已经存在键，则之前的值将仍然保持相同，并且您将获得响应 NOT_STORED
命令格式:
add key flag expiretime bytes
data
示例:
add userId 0 0 4
1234

查: get
注: get 命令相当简单。您使用一个键来调用 get，如果这个键存在于缓存中，则返回相应的值。如果不存在，则不返回任何内容。
命令格式:
get key
示例:
get userId

改: replace
注: 仅当键已经存在时，replace 命令才会替换缓存中的键。如果缓存中不存在键，那么您将从Memcached 服务器接受到一条 NOT_STORED 响应。
命令格式:
replace key flag expiretime bytes
data
示例:
replace userId 0 0 5    
12345

删: delete key [time]
注: delete 命令用于删除 memcached 中的任何现有值。您将使用一个键调用delete，如果该键存在于缓存中，则删除该值。如果不存在，则返回一条NOT_FOUND 消息。
time参数是指使key失效并在time秒内不允许用此key
命令格式:
delete key
示例:
delete userId

增改: set
注: add表示如果服务器没有保存该关键字的情况下，存储该数据；replace表示在服务器已经拥有该关键字的情况下，替换原有内容。
命令格式:
set key flag expiretime bytes
data
示例:
set userId 0 0 6
123456
```
<br>

#### 3. 其他命令

```
全删: flush_all [time]
time参数是指是所有缓存失效,并在time秒内限制使用删除的key

增减: incr/decr key value
注:value及增减后的结果,都是32位无符号整数
0-2^32-1,
统计命令 stats
```
<img src="http://p07ywvfks.bkt.clouddn.com/Mem-stats.png" class="img-responsive img-rounded" />

```
pid  服务器进程的进程号
uptime  服务器自运行以来的秒数
time  当前服务器上的UNIX时间
version string 服务器的版本字符串
curr_items 当前在服务器上存储的数据项的个数
cmd_get  get命令请求的次数
cmd_set  存储命令请求的次数
get_hits  关键字获取命中的次数
```
<br>

#### 4. Memcached内存管理机制
<img src="http://p07ywvfks.bkt.clouddn.com/Mem-slab.png" class="img-responsive img-rounded" />

```
Memcached利用Slab Allocation机制来分配和管理内存
它按照预先分配的大小，将分配的内存分割成特定长度的内存块，再把尺寸相同的内存块分成组，这些内存块不会释放，可以重复利用。
```
<br>

#### 5. Memcached数据过期方式
5.1 Lazy Expiration
```
memcached内部不会监视记录是否过期，而是在get时查看记录的时间戳，检查记录是否过
期。这种技术被称为lazy（惰性）expiration。
因此，memcached不会在过期监视上耗费CPU时间。
```

5.2 LRU(Least Recently Used 最近最少使用)
```
1.Memcached会优先使用已超时的记录的空间
2.如果Memcached的内存空间不足时，就使用LRU模型从最近未被使用的记录中搜索，并将其空
间分配给新的记录。从缓存的实用角度来看，该模型十分理想。
```

















