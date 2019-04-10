---
layout:     post
title:      "TreeRangeMap"
subtitle:   "怎样范围查询缓存"
date:       2019-04-09
author:     "ALID"
header-img: "img/post-bg-digital-native.jpg"
catalog: true
tags:
    - case
    - 数据结构
    - guava
    - java
---

## base
`Range` guava提供的一个用于区间操作的类，提供了以下几种方法：


方法 | 含义
:--- | :---:
open(C, C) 	|  (a..b)
closed(C, C) 	|  [a..b]
closedOpen(C, C) |  [a..b)
openClosed(C, C) |  (a..b]
greaterThan(C) |  (a..+∞)
atLeast(C) 	|  [a..+∞)
lessThan(C) 	|  (-∞..b)
atMost(C) 	|  (-∞..b]
all() 		|  (-∞..+∞)
contains(C value) 	|  包含
encloses(Range range)	|  range是否包含
isConnected |  range是否可连接上
span(Range range) 	| 获取两个range的并集，如果两个range是两连的，则是其最小range
intersection(Range range)| 如果两个range相连时，返回最大交集，如果不相连时，直接抛出异常

## Case
在一个需求中，我希望通过一个时间查到该时间范围对应的Value，进而希望使用Range。尝试写了一个`map<Range<DateTime>, String>` 主要需要实现`get()`方法。

```java
private String getForRange(
HashMap<Range<DateTime>, String> range, DateTime dateTime) {
    Set<Range<DateTime>> ranges = range.keySet();
    List<Range<DateTime>> keys = ranges.stream()
            .filter(d -> d.contains(dateTime))
            .collect(Collectors.toList());
    if (keys.size() == 0) return null;
    if (keys.size() > 1) 
        throw new IllegalArgumentException("区间重复");
    return range.get(keys.get(0));
}
```

每次自己去遍历如果Map大了以后会很耗时间。谷歌一下发现Guava实现了一个TreeRangeMap的数据结构，进而研读了一下源码。

## TreeRangeMap
> 首先就是`Key-Value`的选择，`key` Guava选择了输入区间的最小值作为key值，`Value`则是一个`AbstractMapEntry<Range<K>, V>` 这个Map的key为输入并经过校验的区间，value为输入的value。

**而想要实现这样的结构需要实现两点：**

1. put的时候不能有相互覆盖的区间
对于插入的情况，Guava采用了后插入的区间覆盖前区间的方法，如果有重叠的情况，则删除前区间的重叠部分，保留新插入的部分。
3. get的时候要高效获取
获取的时候因为是找区间所以不能直接获取，也不能使用想我的test一样使用遍历寻找的方法。`TreeRangeMap`的实现就使用了`TreeMap`的数据结构。可以获得较好的插入和获取时间。

### init

```java
//初始化存储结构
private final NavigableMap<Cut<K>, RangeMapEntry<K, V>> entriesByLowerBound; 
```
Cut<K> 是一个抽象类用于表示开闭区间的最小和最大值(比如(1,2]就是一个左开右闭区间), 具体使用4个继承Cut的内部类 `BelowValue`，`AboveValue`，`BelowAll`，`AboveAll` 来表示

```java
public static <K extends Comparable, V> TreeRangeMap<K, V> create() {
  // 使用Guava常用的模式，通过方法来创建对象
  return new TreeRangeMap<K, V>(); 
}
```

Guava很喜欢使用方法来进行对象的创建,比如Lists,GuavaCache等等

```java
private TreeRangeMap() {
  // 可以看到就是创建了一个 TreeMap
  this.entriesByLowerBound = Maps.newTreeMap(); 
}
```

其实对于RangeMap不只有TreeMap的实现,还有ImmutableRangeMap的实现方式,这里只涉及TreeMap

```java
private static final class RangeMapEntry<K extends Comparable, V> 
    extends AbstractMapEntry<Range<K>, V> { 
  // NavigableMap<Cut<K>, RangeMapEntry<K, V>> 中的Value结构
  // 继承AbstractMapEntry<Range<K>, V>
  private final Range<K> range; 	// key为Range
  private final V value;	

	RangeMapEntry(Cut<K> lowerBound, Cut<K> upperBound, V value) {
  		this(Range.create(lowerBound, upperBound), value);
    }

	RangeMapEntry(Range<K> range, V value) {
  		this.range = range;
  		this.value = value;
	}

	// ...
}
```


### put
```java
public void put(Range<K> range, V value) {
  if (!range.isEmpty()) {
    checkNotNull(value);
    remove(range); // 判断是否有重叠情况，并处理
    // key为区间最小值
    entriesByLowerBound.put(range.lowerBound, 
        new RangeMapEntry<K, V>(range, value));
  }
}
```
前提到了不能存在交集, 如果有交集就需要使用下面的remove方法进行去重.

### remove
```java
// 获取rangeToRemove低点所覆盖已有Range
Map.Entry<Cut<K>, RangeMapEntry<K, V>> mapEntryBelowToTruncate 
    = entriesByLowerBound.lowerEntry(rangeToRemove.lowerBound);
// 如果低点有重叠则进行截取
if (mapEntryBelowToTruncate != null) {
  RangeMapEntry<K, V> rangeMapEntry = mapEntryBelowToTruncate.getValue();
  if (rangeMapEntry.getUpperBound().compareTo(rangeToRemove.lowerBound) > 0) {
    if (rangeMapEntry.getUpperBound().compareTo(rangeToRemove.upperBound) > 0) {
        // 新插入的节点被完全包含了
        putRangeMapEntry(
        // CASE: 
        // 1. PUT [3，10] A
        // 2. PUT [5, 7] B 
        // 再走一步会 [3，10] A + [5,7] B -> (7,10] A
        // 再加上之后的拆分 => [3,5) A | [5,7] B | (7,10] A
        rangeToRemove.upperBound, // 新插入节点的最大值
        rangeMapEntry.getUpperBound(), // 包含新节点的老节点的最大值
        mapEntryBelowToTruncate.getValue().getValue()); // 老节点的value
    }
    // 有交集存在
    putRangeMapEntry(
        rangeMapEntry.getLowerBound(),
        rangeToRemove.lowerBound,
        mapEntryBelowToTruncate.getValue().getValue());
  }
}
```

这里是左端交集的情况,另一端完全一样就不赘述了. 

```java
private void putRangeMapEntry(Cut<K> lowerBound, Cut<K> upperBound, V value) {
  entriesByLowerBound.put(lowerBound, new RangeMapEntry<K, V>(lowerBound, upperBound, value));
}
```

 在Remove方法中用到的方法 就是put了一个新的Entry

### get
```java
public V get(K key) {
  Entry<Range<K>, V> entry = getEntry(key);
  return (entry == null) ? null : entry.getValue();
}

public Entry<Range<K>, V> getEntry(K key) {
  // 使用TreeMap的floorEntry方法查找对应的Key
  Map.Entry<Cut<K>, RangeMapEntry<K, V>> mapEntry =
      entriesByLowerBound.floorEntry(Cut.belowValue(key));
  if (mapEntry != null && mapEntry.getValue().contains(key)) {
    return mapEntry.getValue();
  } else {
    return null;
  }
}
```
