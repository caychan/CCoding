## 单表访问方法

寻找数据的方式

1. 数据在哪？B+树上

1. 怎么找到？

   1. 直接在叶子节点上遍历：也就是全表扫描
   1. 通过索引找数据：
      1. 通过主键
      1. 通过二级索引，再回表

1. 寻找多少数据

   1. 某个固定值，比如`where a='aaa'`

   1. 某个范围，比如`where b>111 and b<222`

   1. 某个条件或null，比如`where a='aaa' or a is null`

      

假设表`t`存在主键索引：`primary key(id）`、`key(idx_a_b_c)`、`key(create_time)`

实际上分为这么几类：

1. const

   1. 根据主键索引直接获得数据
   1. select * from t where id=1;

1. ref

   1. 根据二级索引直接获得数据，不管是否需要回表
   1. select * from t where a = 'hello'
   1. select a,b,c from t where a = 'hello'

1. ref_null

   1. 根据二级索引查询数据，并且还需要查询null列
   1. select * from t where a='111' or a is null;
   1. 如果改为select * from t where a<'111' or a is null; 则type=range；

1. range

   1. 根据主键索引或二级索引查找某个范围
   1. select * from t where id>1 and id<10;
   1. select * from t where create_time>'2021-01-01' and create_tiem<'2021-01-02';

1. index

   1. 全索引扫描，只要扫描索引就可以得到数据，不需要回表
   1. select a,b,c from t where b='world';

1. all

   1. 全表扫描
   1. select * from t where d='123';
   1. select * from t where c='111';
   1. select a,b,c,d from  where b='world';

   