---
layout:     post
title:      门外的那条狗
category: blog

---
## FDTemplateLayoutCell 代码走读

最近组内有人使用了这个tableview算cell高库，一看都是自动算高的，让人感觉很不错，故走读代码，研究其具体运行方式。

[框架地址](https://github.com/forkingdog/UITableView-FDTemplateLayoutCell)

[作者之一](http://blog.sunnyxx.com/2015/05/17/cell-height-calculation/)

首先在博客中提到的使用RunLoop空闲时间执行预缓存任务，这一块代码并没有找到，估计是我看的这个版本已经去掉了这个功能了。

```
- UITableView+FDIndexPathHeightCache.h
- UITableView+FDIndexPathHeightCache.m
- UITableView+FDKeyedHeightCache.h
- UITableView+FDKeyedHeightCache.m
- UITableView+FDTemplateLayoutCell.h
- UITableView+FDTemplateLayoutCell.m
- UITableView+FDTemplateLayoutCellDebug.h
- UITableView+FDTemplateLayoutCellDebug.m
```

我重点关注的几个类

UITableView+FDIndexPathHeightCache 这里是通过NSIndexPath来存储算出来的高的，简单说就是创建了一个二维数组，将数据进行了填充。

UITableView+FDKeyedHeightCache 这里是cell通过key值进行缓存高度的类，和上面实现的功能完全一样，但是做法不同，就是要求每个cell都提供一个identifier来充当key值，这个就比较简单了。事实上这一块代码量并不多。

UITableView+FDTemplateLayoutCell 这里提供计算高度的方法，以及对外的接口，以及封装使用哪种缓存的方式。

了解这个框架，只需要关注几个点，1.高度是怎样算出来的，2，高度是怎样缓存的

高度算出的方法主要是在这里：

```
- (CGFloat)fd_systemFittingHeightForConfiguratedCell:(UITableViewCell *)cell；

关键代码是：

if (cell.accessoryView) {
accessroyWidth = 16 + CGRectGetWidth(cell.accessoryView.frame);
} else {
static const CGFloat systemAccessoryWidths[] = {
[UITableViewCellAccessoryNone] = 0,
[UITableViewCellAccessoryDisclosureIndicator] = 34,
[UITableViewCellAccessoryDetailDisclosureButton] = 68,
[UITableViewCellAccessoryCheckmark] = 40,
[UITableViewCellAccessoryDetailButton] = 48
};
accessroyWidth = systemAccessoryWidths[cell.accessoryType];
}
contentViewWidth -= accessroyWidth;
```

以及

```
fittingHeight = [cell sizeThatFits:CGSizeMake(contentViewWidth, 0)].height;
```

在iOS10.2以上还需要将cell的contentView进行一次强制edge布局，便于最后能够正确的调用sizeThatFits得出高度。

但是，事实上，高度只需要算出一次就可以缓存起来，是不需要反复运算的，所以这里出现了存储的两种方式，1.使用indexpaht，2.使用key-value.这里使用indexpaht比较复杂，主要分为如下几个步骤
预先:首先生成横屏和竖屏两个数组，
获取，赋值：使用这三个方法：

```
- (BOOL)existsHeightAtIndexPath:(NSIndexPath *)indexPath {
[self buildCachesAtIndexPathsIfNeeded:@[indexPath]];
NSNumber *number = self.heightsBySectionForCurrentOrientation[indexPath.section][indexPath.row];
return ![number isEqualToNumber:@-1];
}

- (void)cacheHeight:(CGFloat)height byIndexPath:(NSIndexPath *)indexPath {
self.automaticallyInvalidateEnabled = YES;//save
[self buildCachesAtIndexPathsIfNeeded:@[indexPath]];
self.heightsBySectionForCurrentOrientation[indexPath.section][indexPath.row] = @(height);
}

- (CGFloat)heightForIndexPath:(NSIndexPath *)indexPath {
[self buildCachesAtIndexPathsIfNeeded:@[indexPath]];
NSNumber *number = self.heightsBySectionForCurrentOrientation[indexPath.section][indexPath.row];
#if CGFLOAT_IS_DOUBLE
return number.doubleValue;
#else
return number.floatValue;
#endif
}
```

代码不难懂，这里需要注意的是，每次都要进行一次[self buildCachesAtIndexPathsIfNeeded:@[indexPath]]，这个就是创建占位数据-1，具体就是实时的创建对应cell的占位数据(在二维数组中生成对应的占位数据)：

```
- (void)buildSectionsIfNeeded:(NSInteger)targetSection {
[self enumerateAllOrientationsUsingBlock:^(FDIndexPathHeightsBySection *heightsBySection) {
for (NSInteger section = 0; section <= targetSection; ++section) {
if (section >= heightsBySection.count) {
heightsBySection[section] = [NSMutableArray array];
}
}
}];
}

- (void)buildRowsIfNeeded:(NSInteger)targetRow inExistSection:(NSInteger)section {
[self enumerateAllOrientationsUsingBlock:^(FDIndexPathHeightsBySection *heightsBySection) {
NSMutableArray<NSNumber *> *heightsByRow = heightsBySection[section];
for (NSInteger row = 0; row <= targetRow; ++row) {
if (row >= heightsByRow.count) {
heightsByRow[row] = @-1;
}
}
}];
}
```

通过上面这五个函数，数据就已经做到了缓存了，建表，缓存，获取，事实上我们的tableview可能会经常有变化的，用户删除了cell，那么我们对应的算高也要更新，这里又要更新一波二维数组，实际的做法是，利用runtime拦截系统方法。

```
+ (void)load {
// All methods that trigger height cache's invalidation
SEL selectors[] = {
@selector(reloadData),
@selector(insertSections:withRowAnimation:),
@selector(deleteSections:withRowAnimation:),
@selector(reloadSections:withRowAnimation:),
@selector(moveSection:toSection:),
@selector(insertRowsAtIndexPaths:withRowAnimation:),
@selector(deleteRowsAtIndexPaths:withRowAnimation:),
@selector(reloadRowsAtIndexPaths:withRowAnimation:),
@selector(moveRowAtIndexPath:toIndexPath:)
};

for (NSUInteger index = 0; index < sizeof(selectors) / sizeof(SEL); ++index) {
SEL originalSelector = selectors[index];
SEL swizzledSelector = NSSelectorFromString([@"fd_" stringByAppendingString:NSStringFromSelector(originalSelector)]);
Method originalMethod = class_getInstanceMethod(self, originalSelector);
Method swizzledMethod = class_getInstanceMethod(self, swizzledSelector);
method_exchangeImplementations(originalMethod, swizzledMethod);
}
}
```

看到没，这些系统方法已经被替换了前缀为fd_的对应函数了。

比如说

```
- (void)fd_reloadData {
if (self.fd_indexPathHeightCache.automaticallyInvalidateEnabled) {
[self.fd_indexPathHeightCache enumerateAllOrientationsUsingBlock:^(FDIndexPathHeightsBySection *heightsBySection) {
[heightsBySection removeAllObjects];
}];
}
FDPrimaryCall([self fd_reloadData];);
}
```

这里就是reloadData方法触发时调用，将横竖屏的数据前部清空。
再比如：

```
- (void)fd_insertSections:(NSIndexSet *)sections withRowAnimation:(UITableViewRowAnimation)animation {
if (self.fd_indexPathHeightCache.automaticallyInvalidateEnabled) {
[sections enumerateIndexesUsingBlock:^(NSUInteger section, BOOL *stop) {
[self.fd_indexPathHeightCache buildSectionsIfNeeded:section];
[self.fd_indexPathHeightCache enumerateAllOrientationsUsingBlock:^(FDIndexPathHeightsBySection *heightsBySection) {
[heightsBySection insertObject:[NSMutableArray array] atIndex:section];
}];
}];
}
FDPrimaryCall([self fd_insertSections:sections withRowAnimation:animation];);
}
```
这里就是将insert方法替换，先在对于的二维数组中插入可变数组。然后再在数组中填充高度。

这就是使用二维数组的整个逻辑。

使用key-value的形式就更加简单了：

```
- (BOOL)existsHeightForKey:(id<NSCopying>)key {
NSNumber *number = self.mutableHeightsByKeyForCurrentOrientation[key];
return number && ![number isEqualToNumber:@-1];
}

- (void)cacheHeight:(CGFloat)height byKey:(id<NSCopying>)key {
self.mutableHeightsByKeyForCurrentOrientation[key] = @(height);
}

- (CGFloat)heightForKey:(id<NSCopying>)key {
#if CGFLOAT_IS_DOUBLE
return [self.mutableHeightsByKeyForCurrentOrientation[key] doubleValue];
#else
return [self.mutableHeightsByKeyForCurrentOrientation[key] floatValue];
#endif
}

- (void)invalidateHeightForKey:(id<NSCopying>)key {
[self.mutableHeightsByKeyForPortrait removeObjectForKey:key];
[self.mutableHeightsByKeyForLandscape removeObjectForKey:key];
}

- (void)invalidateAllHeightCache {
[self.mutableHeightsByKeyForPortrait removeAllObjects];
[self.mutableHeightsByKeyForLandscape removeAllObjects];
}
```

由于不存在删除key的情况，所以不需要考虑拦截系统操作。
