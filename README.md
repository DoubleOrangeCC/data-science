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

使用datagrip将保存为csv格式的数据集导入到MySQL数据库中
数据预处理:
1.数据导入
创建新数据库userbehavior
create database userbehavior;
右键userbehavior数据库--导入/导出--从文件导入数据--选择csv文件路径--确认导入
2.修改更新表字段
源数据表字段不明确,修改名称和数据类型并且添加汉语注释
用户ID/商品ID/商品类目ID的数据类型保留,均为默认的INT
用户行为为固定的四种行为:点击,购买,加入购物车,收藏,因此修改用户行为为枚举类型enum
数据集中关于时间的数据是以时间戳的形式存在的,需要转换成日期格式,因此修改日期为字符串类型varchar
alter table user_behavior
    change C1 user_id int comment '用户ID';
alter table user_behavior
    change C2 item_id int comment '商品ID';
alter table user_behavior
    change C3 `category_id` int comment '商品类目ID';
alter table user_behavior
    change C4 behavior enum ('pv', 'buy', 'cart', 'fav') comment '用户行为';
alter table user_behavior
    change C5 datetime varchar(50) comment '行为日期';

该数据集包含100万个用户的行为数据,一共1亿行,由于电脑性能有限,故取10w个用户的数据建立新表进行分析
create table userbehavior_10w as
with cte as (select distinct user_id from userbehavior order by user_id limit 100000)
select *
from userbehavior
where user_id in (select user_id from cte);

添加日期/小时字段为后续数据分析做准备
alter table userbehavior_10w
    add column date varchar(20);
alter table userbehavior_10w
    add column hour varchar(10);
将时间戳转化为日期
update userbehavior_10w
set datetime = from_unixtime(datetime, '%Y-%m-%d %H:%m:%s');
将日期拆分到日和小时字段
update userbehavior_10w
set date = date_format(datetime, '%Y-%m-%d');
update userbehavior_10w
set hour = date_format(datetime, '%H');

3.处理缺失值
通过对比总行数和字段非null值行数可以发现日期有29个缺失值
select count(*) as '总计数',
       count(user_id),
       count(item_id),
       count(category_id),
       count(behavior),
       count(datetime)
from userbehavior_10w;
进一步统计null值为user_id34939与user_id49210的两个用户
select *
from userbehavior_10w
where user_id is null
   or item_id is null
   or category_id is null
   or behavior is null
   or datetime is null;
以用户为基数计算缺失率=2/100000 缺失率非常低, 因此不考虑填充而选择删除
delete
from userbehavior_10w
where datetime is null;
删除后总行数与各字段非null值行数相同

4.处理重复值
查询重复值,大于500行,重复数据过多
select *
from userbehavior_10w
group by user_id, item_id, category_id, behavior, datetime
having count(*) > 1;
对源数据进行去重处理
新增一个自增字段create_id作为主键,保留重复数据中最小create_id,删除其他所有重复数据
alter table userbehavior_10w
    add column create_id bigint unsigned not null auto_increment primary key first;
with duplicates as (select user_id, item_id, category_id, behavior, datetime, min(create_id) as main_id
                    from userbehavior_10w
                    group by 1, 2, 3, 4, 5
                    having count(*) > 1)
delete u
from userbehavior_10w u
         join duplicates d on d.user_id = u.user_id and d.item_id = u.item_id and d.category_id = u.category_id and
                              d.datetime = u.datetime
where u.create_id != d.main_id;
删除自增字段
alter table userbehavior_10w
    drop create_id;

5.处理异常值
数据集介绍虽然显示时间区间为2017年11月25日至2017年12月3日,但查询发现仍有不在此区间的数据(此处定义为异常数据)
select min(datetime) as min, max(datetime) as max
from userbehavior_10w;
with cte as (select *
             from userbehavior_10w
             where datetime > '2017-12-03 23:59:59'
                or datetime < '2017-11-25 00:00:00'
             order by datetime)
select count(1) as cnt
from cte;
删除异常数据
delete
from userbehavior_10w
where datetime > '2017-12-03 23:59:59'
   or datetime < '2017-11-25 00:00:00';


数据总览
select count(distinct user_id)     as '总用户数',
       count(distinct item_id)     as '总商品数',
       count(distinct category_id) as '总商品类目数',
       count(behavior)             as '行为总次数'
from userbehavior_10w;

用户生命周期分析

获取情况

日新增用户数(由于数据集时间只包含特定日期段,因此假设'2017-11-25'为所有数据的初始日期,
with cte as (select user_id, min(date) as 'fir_d' from userbehavior_10w group by user_id)
select count(user_id) as '新增用户数',  fir_d as '日期' from cte group by fir_d order by 2;

活跃情况
每日页面访问量(点击量)PV
每日独立访客UV
平均每人页面访问量PV/UV
select date,
       sum(case when behavior = 'pv' then 1 else 0 end)                           as 'PV',
       count(distinct user_id)                                                    as 'UV',
       sum(case when behavior = 'pv' then 1 else 0 end) / count(distinct user_id) as 'PV/UV'
from userbehavior_10w
group by date
order by date;

留存情况

基于活跃用户的留存率:
次日留存率
with cte as (select u1.date, count(distinct u1.user_id) as cnt1
             from userbehavior_10w u1
                      join userbehavior_10w u2 on datediff(u2.date, u1.date) = 1 and u1.user_id = u2.user_id
             group by u1.date),
     cte2 as (select date, count(distinct user_id) as cnt2 from userbehavior_10w group by date)
select cte.date, cnt1 / cnt2 as "retention1"
from cte
         join cte2 on cte.date = cte2.date
order by 1;


WITH user_activity AS (
  SELECT
    user_id,
    date,
    LEAD(date) OVER (PARTITION BY user_id ORDER BY date) AS next_date
  FROM userbehavior_10w
)
SELECT
  date,
  COUNT(DISTINCT user_id) AS active_users,
  SUM(CASE
    WHEN next_date = date + INTERVAL 1 DAY THEN 1
  END) AS retained_users,
  ROUND(
   SUM(DISTINCT CASE
      WHEN next_date = date + INTERVAL 1 DAY THEN 1
    END) / COUNT(DISTINCT user_id),
    4
  ) AS retention1
FROM user_activity
GROUP BY date
ORDER BY date;
#优化版:使用lead()窗口函数与interval + 1 day 避免复杂的join

终极优化:
WITH user_activity AS (
  SELECT
    user_id,
    date,
    LEAD(date, 1) OVER (PARTITION BY user_id ORDER BY date) AS next_date
  FROM userbehavior_10w
  GROUP BY user_id, date  -- 去重确保每日只计一次
)
SELECT
  date,
  COUNT(DISTINCT user_id) AS active_users,
  COUNT(DISTINCT CASE
    WHEN next_date = date + INTERVAL 1 DAY THEN user_id
  END) AS retained_users,
  COUNT(DISTINCT CASE
    WHEN next_date = date + INTERVAL 1 DAY THEN user_id
  END) / COUNT(DISTINCT user_id) AS retention1
FROM user_activity
GROUP BY date
ORDER BY date;



7日留存率
with cte as (select u1.date, count(distinct u1.user_id) as cnt1
             from userbehavior_10w u1
                      join userbehavior_10w u2 on datediff(u2.date, u1.date) = 7 and u1.user_id = u2.user_id
             group by u1.date),
     cte2 as (select date, count(distinct user_id) as cnt2 from userbehavior_10w group by date)
select cte.date, cnt1 / cnt2 as "retention1"
from cte
         join cte2 on cte.date = cte2.date
order by 1;

基于新增用户的留存率:
次日留存率
with cte as (select user_id, min(date) as date from userbehavior_10w group by user_id),
     cte2 as (select cte.date, count(cte.user_id) as cnt
              from cte
                       join userbehavior_10w u on datediff(u.date, cte.date) = 1 and cte.user_id = u.user_id
              group by date),
     cte3 as (select date, count(user_id) as cnt2 from cte group by date)
select cte3.date, cnt / cnt2 as '次日留存率'
from cte3
         left join cte2 on cte2.date = cte3.date
order by cte3.date;

变现情况
购买用户数、付费率、平均每用户收入（ARPU）、客户生命周期价值（LTV）。

流失情况
用户流失率、用户回流率

跳失用户
SELECT user_id from userbehavior_10w group by user_id having count(*) = 1;

