---
layout: post
title: "iOS中KV数据库LevelDB的编译和使用"
date: 2016-07-10 12:15
comments: true
tags: iOS
---

LevelDB是Google公司重量级工程师[`Jeff Dean`](http://research.google.com/people/jeff/index.html)和[`Sanjay Ghemawat`](http://research.google.com/people/sanjay/index.html)发起的开源项目。LevelDB是一个持久化存储的KV系统，和Redis这种内存型的KV系统不同，LevelDB不会像Redis一样狂吃内存，而是将大部分数据存储到磁盘上。目前能够支持billion级别的数据量，在这个数量级别下还有着非常高的性能，主要归功于它的良好的设计。

LevelDB开源并托管在GitHub上，项目的地址是：[https://github.com/google/leveldb](https://github.com/google/leveldb)。

有个来自LevelDB官方对LevelDB、TreeDB和SQLite3进行性能对比分析的测试，测试结果如下图所示：

![leveldb-speed.png](/images/ios-leveldb/leveldb-speed.png)

结果显示，在顺序读写和随机写上，LevelDB 在性能上都遥遥领先。

###编译iOS静态库

####1.下载代码到本地

`git clone https://github.com/google/leveldb.git`

####2.编译项目代码

```
cd leveldb
CXXFLAGS=-stdlib=libc++ make PLATFORM=IOS
```
如果出现如下报错信息：

```
c++ -stdlib=libc++ -I. -I./include -std=c++0x  -DOS_MACOSX -DLEVELDB_PLATFORM_POSIX -DLEVELDB_ATOMIC_PRESENT -O2 -DNDEBUG -fPIC -c db/builder.cc -o /db/builder.o
error: unable to open output file '/db/builder.o': 'Operation not permitted'
1 error generated.
make: *** [/db/builder.o] Error 1
```
使用`sudo CXXFLAGS=-stdlib=libc++ make PLATFORM=IOS`这行命令即可。

说明：

（1）编译完成之后在`out-ios-universal`这个目录下面会自动生成`libleveldb.a`和`libmemenv.a`两个文件。

（2）可以用`lipo -info libleveldb.a`检测生成的静态库支持的架构情况。默认支持`armv6 armv7 armv7s i386 x86_64 arm64`所有的架构

（3）头文件在`include`目录下面，后面会用到

###在iOS中使用LevelDB

LevelDB提供的是C++的API，可以在`https://rawgit.com/google/leveldb/master/doc/index.html`这里查到具体的使用说明。使用C++确实是不太方便幸好有大神将这些接口使用Objective-C进行了一下包装，使用方式和NSUserDefaults一致，可以参考[《轻量级的KV数据库LevelDB在Objective-C上的应用》](http://www.tanhao.me/pieces/1397.html)这篇文章。

为了便于使用和项目集成我将这个和编译好的静态库放在了一起做成一个库。可以直接使用CocoaPods进行集成。

```
pod 'leveled-pd', :git => 'https://github.com/hhtczengjing/leveldb-pd.git'
```

(1)初始化数据库

```
NSString *docPath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) lastObject];
NSString *pageDBPath = [docPath stringByAppendingPathComponent:@"my_leveldb.ldb"];
THLevelDB *myLevelDB = [THLevelDB levelDBWithPath:pageDBPath];
```

(2)存储数据

```
[myLevelDB setString:@"hello world" forKey:@"username"];
```

(3)读取数据

```
NSString *str = [myLevelDB stringForKey:@"username"];
```

没错就是这样方便。

###参考资料

1.[《Github Project Home》](https://github.com/google/leveldb)

2.[《LevelDB library documentation》](https://rawgit.com/google/leveldb/master/doc/index.html)

3.[《轻量级的KV数据库LevelDB在Objective-C上的应用》](http://www.tanhao.me/pieces/1397.html)

4.[《编译leveldb for iOS》](http://blog.txx.im/blog/2014/01/20/build-leveldb/)

5.[《LevelDB、TreeDB、SQLite3性能对比测试》](http://blog.nosqlfan.com/html/2819.html)