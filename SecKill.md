## 通过锁实现

### 悲观锁

sychronized





### 乐观锁

#### 1、数据库乐观锁

1、基于MySQL表中的version字段（需要修改数据库的version字段）

```sql
update items
set amount=amount-#{buys} AND version=version+1
where item_id = #{id} and version=#{version}
```



2、MySQL的状态

```sql
update items
set amount=amount-#{buys} 
where item_id = #{id} and amount-buy>0;
```



#### 2、缓存

CAS机制

读取数据-->> 操作数据 -->> 比较（更新或取消）





memcached的cas机制

<img src="/Users/bill/Library/Application Support/typora-user-images/image-20200701212736637.png" alt="image-20200701212736637" style="zoom:50%;" />

redis事务的watch（基于CAS）

