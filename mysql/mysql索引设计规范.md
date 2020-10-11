## mysql数据索引失效场景分析



### 1.何为索引

以主键索引为例，每个数据页的页号，还有数据页里最小的主键值放在一起，组成一个索引的目录。



> ![image-20201010141140976](https://tva1.sinaimg.cn/large/007S8ZIlly1gjkgxmo9waj319g0kkds4.jpg)

当索引过多时，索引页也会进行相应的页分裂，在更高的索引层级里，保存了每个索引页和索引页里的最小主键值，最后形成树形数据结构，即B+树(本质是二分查找)。

> ![image-20201010143316278](https://tva1.sinaimg.cn/large/007S8ZIlly1gjkgxu13gvj31b60i6tg1.jpg)

### 2.两种索引

#### （1）聚簇索引

* 叶子节点就是数据页本身，数据增删改时数据库自动维护
* 索引页和数据页之间有指针连接起来
* 同一层级的索引页，互相之间基于指针组成双向链表。

> ![image-20201010143701080](https://tva1.sinaimg.cn/large/007S8ZIlly1gjkgxx349wj319w0kcdoo.jpg)

#### （2）非聚簇索引

* 重新建立一棵B+树，可能发生回表。

* 比如基于"name"字段建立一个索引，新建立的B+树的叶子节点数据页中只存放主键和name字段，若需要其他信息，需要回表查询。

  

### 3.索引冗余的坏处

* 空间：每一个索引即一棵B+树，会耗费磁盘空间。

* 时间： 每棵B+树都要求页内按照值的大小进行排序，页与页之间也存在顺序，下一页的所有值必须大于上一页的所有值。故索引过多时，增删改会触发数据页的大量挪动以及分裂，造成时间过长。

  

### 4.联合索引使用规范

> | id   | class_name | student_name | subject_name | subject_score |
> | :--- | :--------: | :----------: | :----------: | :-----------: |
> | 113  |    2班     |     张三     |     语文     |      67       |
> | 178  |    4班     |     李四     |     英语     |      94       |
> | 225  |    1班     |     王五     |     数学     |      86       |

<center>索引：KEY(class_name,student_name,subject_name)</center>

#### 	①等值匹配原则

​		` select * from table where class_name='1班' and student_name ='王五' and subject_name = '数学'   `    

> 就是你发起的Sql语句里，where条件里的几个字段与联合索引的字段完全一样，而且都是基于等号的等值匹配，这样百分之百会走索引。
>
> 即使where语句里的字段顺序与联合索引中的字段顺序有差异，优化器会优先按联合索引的字段顺序查找。

#### 	

#### 	②最左侧列匹配原则

​		（1）`select * from table where class_name='1班' and student_name ='王五' `    

​		（2）`select * from table where subject_name ='语文' `

​		（3）`select * from table where class_name='1班' and subject_name ='数学' `

> （1）走索引，从左至右匹配。
>
> （2）不走索引，联合索引的B+树里不能跳过前面两个字段直接查询subject_name字段。
>
> （3）只有class_name的值可以在该索引里搜索，而subject_name是无法在该索引中查找的，原因同上。



#### 	④最左前缀匹配原则

​		（1）`select * from table where class_name like '1%'`    

​		（2）`select * from table where class_name like '%班'`

> （1）走索引，因为联合索引的B+树里都是按照class_name排序的，所以已知最左前缀是1，是可以基于索引来查找的。
>
> （2）不走索引，不知道最左前缀，无法根据排序查找数据。



#### 	⑤范围查找规则

​		（1）`select * from table where class_name >'1班' and class_name <'4班'`    

​		（2）`select * from table where class_name >'1班' and class_name <'4班' and student_name > 0`

> （1）走索引，因为联合索引的最下层的数据页都是按顺序组成双向链表的，所以完全可以先找到'1班'对应的数据页，再找到'5班'对应的数据页，两个数据页中间的那些数据页就是符合要求的数据。
>
> （2）只有class_name是可以基于索引进行查找，student_name的范围查询无法用到该索引。
>
> 总结：where语句中如果有范围查询，那只有对联合索引里最左侧的列进行的范围查询才会用到索引。



#### 	⑥等值+范围匹配规则

​		`select * from table where class_name ='1班' and student_name < ? and subject_name < ?`    

> 首先，数据库引擎可以通过class_name在索引里精确定位到一波数据。
>
> 接着这波数据里的student_name都是按顺序排列的，所以 'student_name < ?' 也会基于索引进行查找。
>
> 但是接下来的 'subject_name < ?' 是不会使用索引的。



### 5.索引设计的几种原则



#### 	①针对sql语句中的where、order by、group by条件

> 就是说，你的where条件里要根据哪些字段来筛选数据？order by 要根据哪些字段来排序？group by要根据哪些字段来分组聚合？
>
> 此时你就可以设计一个或者两三个联合索引，每一个联合索引都尽量去包含上你的where、order by、group by里的字段，接着要仔细审查每个sql语句，是不是每个where、order by、group by后面跟的字段顺序，都是某个联合索引的最左侧字段开始的部分字段。
>
> 比如有一个联合索引时INDEX(a,b,c)，如下三条sql均可以命中索引
>
> where a=? and b=?
>
> order by a,b
>
> group by a

#### 	

#### 	②针对基数比较大的字段

> 在建立索引时，要根据列的基数来建立。
>
> 列基数=列中不同的数据/总数据量
>
> 越接近于1表示重复数据越少，越适合建立索引。
>
> 原因：优化器优化成全表扫描取决与使用最好索引查出来的数据是否超过表的30%的数据。

#### 	

#### 	③针对数据量比较大的字段

> 针对类似varchar(255)的字段，可能由于数据过大回应性磁盘空间及IO读写速度，可以针对这个字段的前20个字符建立索引，是谓前缀索引。
>
> 比如KEY my_index(name(20),age,course)，name虽然是varchar(255)类型的字段，但是在实际的索引树中，对name的值仅仅只提取前20个字符进行数据页排序查询。
>
> 但是需要注意的是，前缀索引不适用于order by或者group by！

#### 	

#### 	④针对主键

> 建议主键一定是自增的，别用UUID之类的。
>
> 因为主键自增，起码增删改的时候，聚簇索引不会频繁的分裂，主键值都是有序的。



### 6.索引失效的几种场景



#### 	①数据区分度不足30%

> 当表的索引被查询，会使用最好的索引，除非优化器使用全表扫描更有效。
>
> 优化器优化成全表扫描取决与使用最好索引查出来的数据是否超过表的30%的数据。
>
> 在建立索引的时候，要根据列基数来建立。列基数=列中不同的数据/除以总数据。
>
> 越接近1表示重复数据越少，越适合建立索引。

#### 	

#### 	②函数操作

> `select * from table where date(c) = '2019-05-21'`-------不走索引
>
> `select * from table where c>=? and c <= ?`-------走索引

#### 	

#### 	③隐式转换

> 如varchar不加单引号的话可能会自动转换为int型，使索引无效，产生全表扫描。
>
> `select user_name,tel from table where tel = 111`--------不走索引
>
>  `select user_name,tel from table where tel = '111'`--------走索引
>
> 如果列类型是字符串，那一定要在条件中将数据使用引号引用起来,否则不使用索引
>
> 字符型字段为数字时在where条件里不添加引号，也不会命中索引

#### 	

#### 	④模糊查询

> `select * from table where a like '%111%'`-------不走索引
>
>  `select * from table where a like '111%'`-------走索引

#### 	

#### 	⑤计算操作

> `select * from table where b-1=1000`-------不走索引
>
> `select * from table where b=1000+1`-------走索引

#### 	

#### 	⑥范围查询

> `select * from table where b>=1 and b<=20000`-------数据量过大时不走索引
>
> 可以分段查询进行优化

### 	

### 	⑦or的使用

> 如果条件中有or，即使其中有条件带索引也不会使用(这也是为什么尽量少用or的原因)
>
> **![img](https://tva1.sinaimg.cn/large/007S8ZIlly1gjkgy5fdndj30ev0cfwel.jpg)**
>
> 注意：要想使用or，又想让索引生效，只能将or条件中的每个列都加上索引

#### 	

#### 	⑧not in ,not exist不使用索引

#### 	

#### 	⑨null的使用

> 使用联合索引时，is not null只要在建立的索引列（不分先后）都会走索引。
>
> 使用in null时，必须要和建立索引第一列一起使用时，才会命中索引。
>
> 当建立索引最左列的条件是is null 时,其他建立索引的列可以是is null（但必须在所有列 都满足is null的时候）,或者=一个值，都会走索引。
>
> 当建立索引的最左列是=一个值时,其他索引列可以是任何情况（包括is null 或者=一个值），都会走索引。

#### 	

#### 	⑩对小表查询，数据量过少，不走索引

#### 	

#### 	⑪join操作

> 在JOIN操作中（需要从多个数据表提取数据时),MYSQL只有在\**主键和外键的数据类型相同时\**才能使用索引，否则即使建立了,索引也不会使用。



