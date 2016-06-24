---
layout:     post
title:      CocoaLumberjack框架学习
category: project
description: 开源日志系统的介绍
---

## 关于日志系统
CocoaLumberjack是一个iOS/OSX 下的一个日志系统，据说比原生的NSLog要快很多倍，我这段时间正好要用上，走读了一下代码，仅作记录。

## 简单介绍
[CocoaLumberjack](https://github.com/CocoaLumberjack/CocoaLumberjack)是一个扩展性较好的一个日志开源框架，以总DDLog为对面接口，内部采用了基于不同的流向的logger来拆分具体的实现，用户可以仅仅使用其中某一个logger来记录，也可以多个logger同时记录，甚至自己注册一个logger来达到自己的目的，达到记录日志的功能。

## 基本框架
 @ DDLog（基础框架）
  
 @ DDASLLogger（发送到苹果的日志系统，他们显示到控制台）
 
 @DDTTYLoyger （发送日志语句到控制台）将 log 发送给 Xcode 的控制台
 
 @DDFIleLoger （把输出信息写进文件中）将 log 写入本地文件
 
 这3个是日志的输出流器，用户可以自行决定是否需要注册这些，个人认为DDASLLogger是没有必要的，因为这个实际上输出到苹果的/var/log/messagelog下面去的，所以这个可以不注册
 
<img src="http://7xr0og.com1.z0.glb.clouddn.com/DDLog1.png" width="70%" height="70%">
 
 
 所有的log都通过DDLog进行分发，在DDLog这里已经将整个message日志进行了打包，并且区分同步异步，然后丢到DDLog下的GCD队列中，队列会将日志丢到各个解析器中去解析。每个 Logger 处理收到的 log 也是在它们自己的 GCD队列下（loggingQueue）做的，它们询问其下的 Formatter，获取 Log 消息格式，然后最终根据 Logger 完成最终的输出。
 
 <img src="http://7xr0og.com1.z0.glb.clouddn.com/DDLog2.png" width="70%" height="70%">
 
 
 整体架构：
 
 <img src="http://7xr0og.com1.z0.glb.clouddn.com/DDlog3.png" width="70%" height="70%">
 
## 代码走读
经过宏解析到

```
- (void)queueLogMessage:(DDLogMessage *)logMessage asynchronously:(BOOL)asyncFlag 
```

在这里做了这么几件事，一个是讲日志入DDLog的队列，而这个队列长度有限，默认1000，这里做了一个信号量，主要是为了保证队列最大数不会被超出。一般情况下1000条日志确实不会在同一时间出现，而且只有ERROR等级的日志才会被同步输出，其他等级的日志异步输出。我测试了一下，将队列改小，且换成ERROR等级日志，那么如果超出的数过大，确实会造成当前的打印日志的线程阻塞。

如下代码做了修改，测试极端情况


```
- (void)queueLogMessage:(DDLogMessage *)logMessage asynchronously:(BOOL)asyncFlag {

    static int i = 0;
    i++;
    NSLog(@"111 ************ %@ %d",_queueSemaphore,i);
    dispatch_semaphore_wait(_queueSemaphore, DISPATCH_TIME_FOREVER);

    // We've now sure we won't overflow the queue.
    // It is time to queue our log message.
    NSLog(@"222 ************ %@ %d",_queueSemaphore,i);
    dispatch_block_t logBlock = ^{
        @autoreleasepool {
            NSLog(@"wode xiancheng shi  %@",[NSThread currentThread]);
            sleep(2);
            [self lt_log:logMessage];
        }
    };

    if (asyncFlag) {
        dispatch_async(_loggingQueue, logBlock);
        
    } else {
        dispatch_sync(_loggingQueue, logBlock);
        
    }
    NSLog(@"333 ************ %@ %d",_queueSemaphore,i);
}
```

2016-06-23 11:38:02.808 RuiFuTech[26157:1191052] 111 ************ <OS_dispatch_semaphore: 0x7ff449f2dcd0> 5

2016-06-23 11:38:02.809 RuiFuTech[26157:1191052] 222 ************ <OS_dispatch_semaphore: 0x7ff449f2dcd0> 5

2016-06-23 11:38:02.809 RuiFuTech[26157:1191052] 333 ************ <OS_dispatch_semaphore: 0x7ff449f2dcd0> 5

2016-06-23 11:38:02.809 RuiFuTech[26157:1191052] 111 ************ <OS_dispatch_semaphore: 0x7ff449f2dcd0> 6

2016-06-23 11:38:02.805 [ReOrient]: ----------------------- first ---------------------

2016-06-23 11:38:04.815 RuiFuTech[26157:1191052] 222 ************ <OS_dispatch_semaphore: 0x7ff449f2dcd0> 6

2016-06-23 11:38:04.815 RuiFuTech[26157:1191089] wode xiancheng shi  <NSThread: 0x7ff449c1c7f0>{number = 2, name = (null)}

2016-06-23 11:38:04.816 RuiFuTech[26157:1191052] 333 ************ <OS_dispatch_semaphore: 0x7ff449f2dcd0> 6

2016-06-23 11:38:04.816 RuiFuTech[26157:1191052] 111 ************ <OS_dispatch_semaphore: 0x7ff449f2dcd0> 7

分析：
队列总数为5，第6条日志被卡住2秒钟，直到第一个日志输出成功后，第6条日志加入队列，第七条日志再被卡住。注意这里是DDLog用的是串行队列。看来作者是笃定了后续各个解析器logger够快，而且不会有我这样的测试。


在下一层：

```
- (void)lt_log:(DDLogMessage *)logMessage
```

这里判断多核处理器后，遍历之前注册的loggers，判断message的等级是否达到logger的输出等级。

```
 if (!(logMessage->_flag & loggerNode->_level)) {
                continue;
 }
```
说到这里，有必要将这个枚举贴出来：
我们一般用位移来做这个，但是这样写，简直是清晰明了高大上。

```
typedef NS_ENUM(NSUInteger, DDLogLevel){
    /**
     *  No logs
     */
    DDLogLevelOff       = 0,
    
    /**
     *  Error logs only        0x1
     */
    DDLogLevelError     = (DDLogFlagError),     
    
    /**
     *  Error and warning logs   0x11
     */
    DDLogLevelWarning   = (DDLogLevelError   | DDLogFlagWarning),
    
    /**
     *  Error, warning and info logs 0x111
     */
    DDLogLevelInfo      = (DDLogLevelWarning | DDLogFlagInfo),
    
    /**
     *  Error, warning, info and debug logs 
     */
    DDLogLevelDebug     = (DDLogLevelInfo    | DDLogFlagDebug),
    
    /**
     *  Error, warning, info, debug and verbose logs
     */
    DDLogLevelVerbose   = (DDLogLevelDebug   | DDLogFlagVerbose),
    
    /**
     *  All logs (1...11111)
     */
    DDLogLevelAll       = NSUIntegerMax
};

```

之后使用GCD的group做同步，即要所有的logger都输出完这个message后才能退出这个队列。

```
dispatch_group_async(_loggingGroup, loggerNode->_loggerQueue, ^{ @autoreleasepool {
                [loggerNode->_logger logMessage:logMessage];
            } });
   dispatch_group_wait(_loggingGroup, DISPATCH_TIME_FOREVER);
```


如果在DDFileLogger里面- (void)logMessage:(DDLogMessage *)logMessage 加入延时，发现调用全部都被卡住，这里group会等待全部输出完成后才进行下一步,所以我在想，如果是高并发的日志输出的话，File的输出肯定是要慢于控制台的，那么IO高的情况下，全部日志都会被一点点的占满之前的全局GCD同步队列，最后队列满了之后再卡住调用日志的线程。不过这个情况实在是不大可能。

```
- (void)logMessage:(DDLogMessage *)logMessage
```

实际上是重载了DDAbstractLogger的方法，其实这里我们自己要写一个logger，同样继承这个就好了。

重点看下DDFileLogger,可以看下currentLogFileHandle和maybeRollLogFileDueToSize，了解文件的切分和定时任务的指派。差不多就是这些

另外对于DDLoggerNode，其实可以认为是对DDLogger的一层封装，这个可以过一下

## 附
实际使用的时候发现了不少不爽的地方，这里做了修改
修改文件名的时区：这个算是Bug？

```
- (NSString *)newLogFileName {
    NSString *appName = [self applicationName];
    NSDate *date = [NSDate date];  
    NSTimeZone *zone = [NSTimeZone systemTimeZone];
    NSInteger interval = [zone secondsFromGMTForDate: date];
    NSDate *localeDate = [date  dateByAddingTimeInterval: interval];
    NSDateFormatter *dateFormatter = [self logFileDateFormatter];
    NSString *formattedDate = [dateFormatter stringFromDate:localeDate];

    return [NSString stringWithFormat:@"%@ %@.log", appName, formattedDate];
}
```

修改log的指定位置：

```
//这里isChangeToLibrary为NO，则这里日志是放到默认的cache里面，如果要放到library里面，则需要
-(DDFileLogger *)fileLogger{
    if (!_fileLogger) {
        if (self.isChangeToLibrary) {
            NSString * applicationDocumentsDirectory = [[[[NSFileManager defaultManager]
                                                          URLsForDirectory:NSDocumentDirectory inDomains:NSUserDomainMask] lastObject] path];
            DDLogFileManagerDefault *documentsFileManager = [[DDLogFileManagerDefault alloc]
                                                             initWithLogsDirectory:applicationDocumentsDirectory];
            _fileLogger = [[DDFileLogger alloc] initWithLogFileManager:documentsFileManager];
        }else{
            _fileLogger = [[DDFileLogger alloc] init];
        }
    }
    return _fileLogger;
}

[DDLog addLogger:self.fileLogger];

```

