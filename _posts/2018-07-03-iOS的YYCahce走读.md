---
layout:     post
title:      iOS下YYCache走读
category: project
description: iOS下YYCache走读
---


## iOS下YYCache走读

YYCache是一个非常优秀的iOS下持久化底层库，是线程安全的，在我公司的持久化存储都是在使用这个库。这几天有时间，好好走读了一下这个库。

### YYThreadSafeDictionary
这是基于NSMutableDictionary的封装，所有set和get方法，都被加了锁。看源码的时候，发现作者封装的挺好，有这么几个地方。


将所有需要被锁包含的地方，用宏进行包含

```
#define LOCK(...) OSSpinLockLock(&_lock); \
__VA_ARGS__; \
OSSpinLockUnlock(&_lock);


- (id)objectForKey:(id)aKey {
    LOCK(id o = [_dic objectForKey:aKey]); return o;
}
```

这样写，非常适合在包装函数的时候，对收尾进行添加操作，如加锁解锁。

另外对于isEqual方法，这里是这么判断的

```objc
- (BOOL)isEqualToDictionary:(NSDictionary *)otherDictionary {
    if (otherDictionary == self) return YES;
    
    if ([otherDictionary isKindOfClass:YYThreadSafeDictionary.class]) {
        YYThreadSafeDictionary *other = (id)otherDictionary;
        BOOL isEqual;
        OSSpinLockLock(&_lock);
        OSSpinLockLock(&other->_lock);
        isEqual = [_dic isEqual:other->_dic];
        OSSpinLockUnlock(&other->_lock);
        OSSpinLockUnlock(&_lock);
        return isEqual;
    }
    return NO;
}
```
总结：YYThreadSafeDictionary是比较好的解决多线程操作同一个字典，是基于NSMutableDictionary的封装。

### YYMemoryCache

YYMemoryCache 使用 LRU(least-recently-used) 算法来驱逐对象；NSCache 的驱逐方式是非确定性的。
YYMemoryCache 提供 age、cost、count 三种方式控制缓存；NSCache 的控制方式是不精确的。
YYMemoryCache 可以配置为在收到内存警告或者 App 进入后台时自动逐出对象。

这是一个内存数据缓存Cache，它只有两个数据结构，_YYLinkedMapNode和_YYLinkedMap。
_YYLinkedMap是一个管理整个链表node的管理结构类，结构：

```
@interface _YYLinkedMap : NSObject {
    @package
    CFMutableDictionaryRef _dic; // do not set object directly 这是数据存放的地方
    NSUInteger _totalCost;
    NSUInteger _totalCount;
    _YYLinkedMapNode *_head; // MRU, do not change it directly
    _YYLinkedMapNode *_tail; // LRU, do not change it directly
    BOOL _releaseOnMainThread;
    BOOL _releaseAsynchronously;
}

@interface _YYLinkedMapNode : NSObject {
    @package
    __unsafe_unretained _YYLinkedMapNode *_prev; // retained by dic
    __unsafe_unretained _YYLinkedMapNode *_next; // retained by dic
    id _key;
    id _value;
    NSUInteger _cost;
    NSTimeInterval _time;
}
@end
```
_head和_tail是指向这个链表结构的头尾，事实上YYMemoryCache使用提供的接口进行操作node以及结构，而不推荐直接使用_YYLinkedMap来控制。
这个结构需要关注的是_totalCost和_totalCount，他是用来管理自动删除节点的一些内部记录值，需要用到的函数是：

```
- (void)_trimInBackground {
    dispatch_async(_queue, ^{
        [self _trimToCost:self->_costLimit];// 缓存开销数量限制，默认无限制，超过限制则会在后台逐出一些对象以满足限制
        [self _trimToCount:self->_countLimit];//缓存对象数量限制，默认无限制，超过限制则会在后台逐出一些对象以满足限制
        [self _trimToAge:self->_ageLimit];// 缓存时间限制，默认无限制，超过限制则会在后台逐出一些对象以满足限制
    });
}

- (void)_trimToCount:(NSUInteger)countLimit {
    BOOL finish = NO;
    pthread_mutex_lock(&_lock);
    if (countLimit == 0) {
        [_lru removeAll];
        finish = YES;
    } else if (_lru->_totalCount <= countLimit) {
        finish = YES;
    }
    pthread_mutex_unlock(&_lock);
    if (finish) return;
    
    NSMutableArray *holder = [NSMutableArray new];
    while (!finish) {
        if (pthread_mutex_trylock(&_lock) == 0) {
            if (_lru->_totalCount > countLimit) {
                _YYLinkedMapNode *node = [_lru removeTailNode];
                if (node) [holder addObject:node];
            } else {
                finish = YES;
            }
            pthread_mutex_unlock(&_lock);
        } else {
            usleep(10 * 1000); //10 ms
        }
    }
    if (holder.count) {
        dispatch_queue_t queue = _lru->_releaseOnMainThread ? dispatch_get_main_queue() : YYMemoryCacheGetReleaseQueue();
        dispatch_async(queue, ^{
            [holder count]; // release in queue
        });
    }
}
```

这个trimInBackground方法每隔5秒(可修改)会执行一轮，用来逐出一些老旧数据。使用的是removeTailNode这个方法，这个方法将最终修改数据结构，删除节点数据，修改cost和count等数据。需要注意的是，加锁的时候，使用pthread_mutex_trylock是非常好的办法，不能加上就休眠一会儿再尝试加锁。

还有个地方，需要注意，就是[holder count]; 这行代码完全没有看明白，计算count就算是释放holder对象了?直到看到github上的解释是：

> holder 会被 block 捕获，随后会在 block 结束后，在对应的 queue 里得到释放。
> 这条语句，只是为了让 node 捕获到 block 去，所以随便发了个消息以避免被编译器优化掉。

CFMutableDictionaryRef _dic 这个数据是存放数据的字典，但是他是Core Function框架的，不是NS框架的，要注意使用这个框架的对象，要手动进行释放。
```
CFMutableDictionaryRef holder = _dic;
CFRelease(holder);
```

另外，当我看到cost这个参数的时候，YYCache提供了两个方法去set，

```
- (void)setObject:(id)object forKey:(id)key withCost:(NSUInteger)cost;
- (void)setObject:(nullable id)object forKey:(id)key;
- (void)setObject:(id)object forKey:(id)key {
    [self setObject:object forKey:key withCost:0];
}
```
就是说，如果用户不传入cost，那么这个值就是0，好奇的我发现NSCache也有相关的说明，如下：
```
The cost value is used to compute a sum encompassing the costs of all the objects in the cache. Typically, the obvious cost is the size of the value in bytes. If that information is not readily available, you should not go through the trouble of trying to compute it, as doing so will drive up the cost of using the cache. Pass in 0 for the cost value if you otherwise have nothing useful to pass, or simply use the setObject:forKey: method, which does not require a cost value to be passed in.
```
cost最直观的应该是这个数据的消耗大小，当然也可以是其他。如果计算这个cost有困难，就不要用了cost来标记数据。
具体的增删改查这些细节，这里就不一一分析了，看代码就好了。

### YYDiskCache
这个东西是持久缓存的实现类，里面包含有实际封装缓存细节的YYKVStorage类。这两个是比较重要的类：
先说一个有用的知识点：
```
- (instancetype)init UNAVAILABLE_ATTRIBUTE;
+ (instancetype)new UNAVAILABLE_ATTRIBUTE;
```
这个代表，这个对象不能用init和new去实例化，不然会报错。

这里主要是操作这个对象：
```
@interface YYKVStorageItem : NSObject
@property (nonatomic, strong) NSString *key;                ///< key
@property (nonatomic, strong) NSData *value;                ///< value
@property (nullable, nonatomic, strong) NSString *filename; ///< filename (nil if inline)
@property (nonatomic) int size;                             ///< value's size in bytes
@property (nonatomic) int modTime;                          ///< modification unix timestamp
@property (nonatomic) int accessTime;                       ///< last access unix timestamp
@property (nullable, nonatomic, strong) NSData *extendedData; ///< extended data (nil if no extended data)
@end
```

这个就是用来描述一个存储对象的结构，其中，filename这个字段是用来标识，存储到文件系统，还是存储到sqlite的，而这个filename是怎么设置的？
```
- (void)setObject:(id<NSCoding>)object forKey:(NSString *)key {
    if (!key) return;
    if (!object) {
        [self removeObjectForKey:key];
        return;
    }
    
    NSData *extendedData = [YYDiskCache getExtendedDataFromObject:object];
    NSData *value = nil;
    if (_customArchiveBlock) {
        value = _customArchiveBlock(object);
    } else {
        @try {
            value = [NSKeyedArchiver archivedDataWithRootObject:object];
        }
        @catch (NSException *exception) {
            // nothing to do...
        }
    }
    if (!value) return;
    NSString *filename = nil;
    if (_kv.type != YYKVStorageTypeSQLite) {
        if (value.length > _inlineThreshold) {
            filename = [self _filenameForKey:key];
        }
    }
    
    Lock();
    [_kv saveItemWithKey:key value:value filename:filename extendedData:extendedData];
    Unlock();
}
```

可以看到是使用了_inlineThreshold这个做判断，这个值默认是(20KB).

进一步跟进去

```
- (BOOL)saveItemWithKey:(NSString *)key value:(NSData *)value filename:(NSString *)filename extendedData:(NSData *)extendedData {
    if (key.length == 0 || value.length == 0) return NO;
    if (_type == YYKVStorageTypeFile && filename.length == 0) {
        return NO;
    }
    
    if (filename.length) {
        if (![self _fileWriteWithName:filename data:value]) {
            return NO;
        }
        if (![self _dbSaveWithKey:key value:value fileName:filename extendedData:extendedData]) {
            [self _fileDeleteWithName:filename];
            return NO;
        }
        return YES;
    } else {
        if (_type != YYKVStorageTypeSQLite) {
            NSString *filename = [self _dbGetFilenameWithKey:key];
            if (filename) {
                [self _fileDeleteWithName:filename];
            }
        }
        return [self _dbSaveWithKey:key value:value fileName:nil extendedData:extendedData];
    }
}

```
差不多可以看出，数据item是先存到数据库sqlite里面去的。至于value这个到底存到哪里，取决于是否大于默认的20KB。get方法与此类似，故不做赘述。