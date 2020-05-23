# [XXX]
一个高性能的内容定向检索引擎，用来向一个请求推送满足预先定义的定向条件的内容

特点  
+ 高性能 毫秒级响应，单机QPS 万级
+ 高可用 主从架构，选举机制
+ 可扩展 内容维护机制天然保证幂等性(DNF和正倒排索引更新机制)
+ 开箱即用 可独立部署，也可作为检索模块迁入应用
+ 功能丰富 支持推送可定制，快速匹配（一旦命中即返回），定向自动过期
+ 可定制 支持内容加载器定制，支持内容缓存扩展

## 架构



## 布尔表达式检索
定向条件都类似于：20-25岁+女性，20-30岁+女性+北京，

这里的定向条件用布尔表达式进行表示
```
DNF_1 = {
    {age ∈ [20, 25] ∩ gender ∈ (女)} 
    ∪
    {age ∈ [20, 30] ∩ gender ∈ (女) ∩ city ∈ (北京)}
}
```
这里的形式即为析取范式（Disjunctive Normal Form，DNF）
每个DNF都可以分解成一个或者多个交集（conjunction），这个例子由两个交集组成
```
CONJ_1 = {age ∈ [20, 25] ∩ gender ∈ (女)}
```
和
```
CONJ_2 = {age ∈ [20, 30] ∩ gender ∈ (女) ∩ city ∈ (北京)}
```
注：同一属性只在一个交集中出现一次。

每个交集进一步分解成一个或者多个赋值集（assignment），上例中的第一个交集由下面两个赋值集组成
```$xslt
ASSG_1 = age ∈ [20, 25]
```
和
```
ASSG_2 = gender ∈ (女)
```

当收到来自如下的一个内容请求：
```
age∈（23）∩ gender ∈ (女) ∩ city ∈ (上海)
```
我们知道它满足 CONJ_1 的所有条件 ASSG_1 和 ASSG_2，则可以向其推送 CONJ_1 的相应内容；而并不满足 CONJ_2 的所有条件，故无需向其推送 CONJ_2 的相应内容


## 设计思想
[xxx]由以下几个组件组成
+ Starter 启动器，负责引擎的启动
+ Loader 文档加载器
+ Fetcher 文档抽取器
+ IndexBuilder 索引创建器
+ IndexSynchronizer 索引同步器
+ Matcher 检索器

在实际实现中，采用正反双层索引结构设计

两层正排索引，用以维护定向条件的更新
+ content-conjunction
+ conjunction-assignment

两层倒排索引，用来做内容检索
+ conjunction-content
+ assignment-conjunction

### 启动过程

### 定向条件维护
当内容的定向条件发生改变时  
+ 根据content-conjunction找到该内容的原始定向conjunctions
+ 从原始conjunctions的conjunction-content中将该内容移除
+ 若某conjunction已无内容，可以将该conjunction从此索引中删除
    ++ 根据conjunction-assignment找到该conjunction的原始定向assignments
    ++ 从原始assignments的assignment-conjunction中将该内容移除
    ++ 若某assignment已无conjunction，可以将该assignment从此索引中删除

### 内容检索过程
实际检索过程中，通过assignment筛选出满足条件的conjunction，再根据conjunction找出满足条件的内容集合  
+ 当请求的特征 sizeof(attrs) < sizeof(conjunction)，该conjunction一定不满足要求，可以先利用这个判断减少计算
+ 计算特征满足的assignments
+ 查找assignment-conjunction获取conjunctions，当conjunction被命中的次数 hit(conjunction) >= sizeof(conjunction)时，该conjunction定向条件被满足
+ 查找conjunction-content，获取推送内容


## 接口规范

### Contents
### Attrs

0. name 名称，缺省同field
0. field 字段，需要匹配的字段
0. type 类型，匹配字段的类型也即操作数的类型
0. operator 操作符，需支持eq、in、bt、gt、ge、lt、le
0. operand 操作数，

```json
{
  "name" : "年龄",
  "field" : "age",
  "type" : "Integer",
  "operator" : "bt",
  "operand" : [20, 25]
}
```
