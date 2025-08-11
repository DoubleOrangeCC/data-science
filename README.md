# 淘宝用户行为数据分析

本项目是基于MySQL和Tableau的对淘宝用户行为的数据分析


## 数据来源与介绍

阿里云天池公共数据集 https://tianchi.aliyun.com/dataset/649

UserBehavior是阿里巴巴提供的一个淘宝用户行为数据集,用于隐式反馈推荐问题的研究。

本数据集包含了2017年11月25日至2017年12月3日之间，有行为的约一百万随机用户的所有行为（行为包括点击、购买、加购、喜欢）。数据集的组织形式和MovieLens-20M类似，即数据集的每一行表示一条用户行为，由用户ID、商品ID、商品类目ID、行为类型和时间戳组成，并以逗号分隔。关于数据集中每一列的详细描述如下：

| 列名称 | 说明 | 
|---|---|
| 用户ID | 整数类型，序列化后的用户ID | 
| 商品ID | 整数类型，序列化后的商品ID | 
| 商品类目ID | 整数类型，序列化后的商品所属类目ID| 
| 行为类型 |字符串，枚举类型，包括('pv', 'buy', 'cart', 'fav')| 
| 时间戳 | 行为发生的时间戳| 

用户行为类型分为四种:

| 行为类型 | 说明 | 
|---|---|
| pv | 商品详情页pv，等价于点击 | 
| buy | 商品购买 | 
| cart | 将商品加入购物车| 
| fav |收藏商品| 

## 数据预处理

### 数据导入
图形化数据库管理工具选择: Datagrip

1. 新建数据库**userbehavior**
2. 右键**userbehavior**
3. 点击**导入/导出**
4. 点击**从文件导入数据** 
5. 选择 `.csv` 文件路径
6. 新建表并命名**userbehavior**
7.  点击 **确认导入**

### 修改表字段

源数据表字段不明确,修改字段名称和数据类型并且添加汉语注释  

关于时间的数据以时间戳的形式存在,需要使用from_unixtime函数转换成指定的日期格式(字符串),因此提前修改字段为字符串类型
```sql
alter table userbehavior
    change c1 user_id int comment '用户ID',
    change c2 item_id int comment '商品ID',
    change c3 category_id int comment '商品类目ID',
    change c4 behavior enum ('pv', 'buy', 'cart', 'fav') comment '用户行为',
    change c5 datetime varchar(50) comment '行为时间';
```
增加日期/小时字段方便后续数据分析 
```sql
alter table userbehavior
    add column date varchar(20),
    add column hour varchar(10);
```
将时间戳更新为指定格式的日期
```sql
update userbehavior
set datetime = from_unixtime(datetime, '%Y-%m-%d %H:%m:%s');
```
将日期拆分为日/小时字段
```sql
update userbehavior
set
    date = date_format(datetime, '%Y-%m-%d'),
    hour = date_format(datetime, '%H');
```

### 处理缺失值

通过对比总行数和字段非null值行数可以发现日期有29个缺失值  
```sql
select count(*) as '总计数',  
  count(user_id),  
  count(item_id),  
  count(category_id),  
  count(behavior),  
  count(datetime)  
from userbehavior_10w;
```
