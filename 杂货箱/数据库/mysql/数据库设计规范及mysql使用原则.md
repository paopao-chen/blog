# 数据库设计规范

## 基本规范
- 命名规范：所有命名（库，表，字段）都使用小写字母加 _ 组合,user_name, user_age
- 表规范：
    - 有相同关系的表可以使用相同的表前缀：'user_'
    - 每个表都要有自增主键，如果有其他唯一标识可以再加 primary 约束，都要有 'add_time'（默认系统时间）
    - 表与表的关联字段名要求尽可能相同
    - 用尽可能少的存储空间来存一个字段的数据，但要求必须够用！
    - 最好给每个字段一个默认值，尽量都不要为 null
    - 不使用外键，保证数据库易改动（由程序保证约束）

## 数据库范式
1. 第一范式：字段值具有原子性，不能再分（所有关系型数据库都满足）
2. 第二范式：一个表必须具有主键，即每行数据都被唯一区分
3. 第三范式：一个表中不能包含其他相关表中非关键信息，即数据表不能有冗余字段（金额=单价*数量，则金额也属于冗余字段）
4. 备注：在设计时，为了满足性能，往往难以满足第三范式，合理的冗余字段可以减少查询，相册表中会添加图片的点击数字段,在相册图片表中也会添加图片的点击数字段


# mysql 使用原则

## 核心原则
- 控制列数量(字段少而精,字段数建议在20以内)
- 不在数据库做运算,cpu计算务必移至业务层
- 拒绝3B(拒绝大sql语句：big sql、拒绝大事物：big transaction、拒绝大批量：big batch)

## 字段设计原则
- 用好数值类型，能用数值就不要用字符串
- 字符转化为数字(能转化的最好转化,同样节约空间、提高查询性能)，可以在后端常量中加上转化约定
- 避免使用NULL字段，添加 NOT NULL 约束，(NULL字段很难查询优化、NULL字段的索引需要额外空间、NULL字段的复合索引无效，可以添加默认值避免null)

## sql原则
- sql语句尽可能简单(一条sql只能在一个cpu运算,大语句拆小语句,减少锁时间,一条大sql可以堵死整个库)
- 不使用 select * (消耗cpu,io,内存,带宽,且这种程序不具有扩展性)
- OR改写为IN(or的效率是n级别)，OR改写为UNION(mysql的索引合并很弱智)

```
    select id from t where phone = ’159′ or name = ‘john’;
    =>
    select id from t where phone=’159′
    union
    select id from t where name=’jonh’
```

- limit高效分页(limit越大，效率越低)
- 使用group by

