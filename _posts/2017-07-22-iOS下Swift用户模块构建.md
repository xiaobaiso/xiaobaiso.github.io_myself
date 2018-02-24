---
layout:     post
title:      iOS下Swift用户模块构建
category: project
description: iOS下Swift用户模块构建
---

## 起因
最近在用Swift写一个视频在线观看的App，涉及到注册登录等用户权限问题，这一块属于App较为核心的问题，我打算用Swift写一个用于中心来管控用户的权限，浏览历史，以及登录态和用户信息，此文用于记录编写用户中心遇到的一些问题。

## 简单介绍
我想做的事情十分简单，我想通过一个Struct来记录用户所有的相关数据，在Struct中有若干方法用于自计算获得更高一层的有效数据，比如获取一个用户的登录态，应该是（上次的登录时间距离现在仍有效 && 持久化的AccessToken存在 && 未收到服务端强踢协议 ），如果这个括号里的bool为真，那么我就认为该用户可以免登陆继续保持登录状态，而这些逻辑运算都是这个Struct的内部方法，只提供运算的结果(是否登录)给上层业务。

那么，我开始就想的很简单，我是这么想的：把这个Struct完整的保存下来，本地持久化，这样就可以保证每次的运行状态一致性，但是有几个问题要关注：

- 升级时struct多余字段以及删除字段和旧版本持久化数据如何保证兼容
- struct要支持NSCoding协议，才能使用归档的方式持久化
- 保存单元的颗粒度太大，会导致持久化频繁

```

protocol Storable {
    associatedtype ItemType
    func localInit(_ key :String) -> ItemType?;
}

extension Storable{
    func localInit(_ key :String) -> ItemType? {
        let localData :AnyObject? = Cache.shareInstance.object(forKey: key)
        return localData as? ItemType ?? nil
    }
}

class UserManager: Storable {
	.....
}

```

这样看确实好了不少，ItemType可以指定不同的数据结构，但是有个问题，我需要添加NSCoding协议或者将其转化为字典，再存储，这样就涉及一个映射的问题。

而颗粒度大的问题就更加麻烦，我有一个字幕的结构体，字幕当前指向总是要变化，每一次变化都要持久化，而持久化就带动了关联的几个子结构体也同步持久化，这样就显得很智障了。



## 换个思路 

事实上，我可以将颗粒度设置的更加细，比如这样：

```
    var phone :String = "" {
        didSet{
            Cache.saveToLocalString(phone, key: "phone_key")
        }
    }
```

get方法就在单例初始化的时候全部取出来，这样做就一口气解决了上面的三个问题，颗粒度最小，原子性操作，基本数据结构可以直接存储

```
protocol Storable {
    func saveToLocalString(_ data:String,key:String)
    func getLocalStringWithKey(_ key:String) -> String?;
}

extension Storable {
    func saveToLocalString(_ data: String, key: String) {
        //这里管理Node的存错
        Storage.sharedInstance.saveString(data, key: key)
    }
    func getLocalStringWithKey(_ key: String) -> String? {
        return Storage.sharedInstance.getString(key)
    }
}

extension String :Storable{
}

```
让String去实现这个协议确实不是最好的方法，但是为了和上层业务层兼容，String这个类型是最大程度避免类型转化问题的，另外这个Key这个字段必须存在，因为协议里面即便可以编写var类型也是计算变量而不是存储变量所以这个key必须往上面一路带。

```
    var checkCode :String = "" {
        didSet{
            checkCode.saveToLocalString(checkCode, key: "checkCode_key")
        }
    }
```
通过观察者即可解决实时存储的问题了