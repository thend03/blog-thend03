---
draft: false
date: 2023-05-22T20:28:54+08:00
title: "clickhouse删除partition"
slug: "delete-clickhouse-partition"
authors: ["since"]
tags: ["clickhouse"]
description : "删除过期partition"
---

clickhouse中有时候过期partition不能自动删除，需要人工干预，手动删除过期的partition,以释放磁盘。
由于partition不支持条件范围删除，所以写了一个shell脚本，循环删除clickhouse集群中的所有分片的过期partition。

## 注意事项
**删除partition的操作不可逆，即执行之后数据删除之后不可恢，执行脚本之前一定要验证好，不要误删数据以免造成故障。**


## 脚本
脚本内容如下
```
#clickHouse 集群连接参数
password="ck_pwd"
cluster="ck_cluster"

# 数据库和表名称
database="your_db"
table="your_table"

# 要删除的分区的日期范围
end_date="20230423"


# 获取所有的host
hosts=$(clickhouse-client --password="$password" --query="SELECT host_name FROM system.clusters WHERE cluster ='$cluster'")

# 循环遍历 ClickHouse 集群中的所有分片，并执行 SQL 语句
while read -r host; do
  echo "Executing SQL on host: $host"
  #clickhouse-client --host="$host" --password="$password" --query="$sql"
  partitions=$(clickhouse-client --host="$host" --password="$password" --query "SELECT DISTINCT partition FROM system.parts WHERE database='$database' AND table='$table' AND partition < '$end_date'")
  while read -r partition; do
    drop_sql="ALTER TABLE $database.$table DROP PARTITION '$partition'"
    echo "executing drop, host:$host, partition:$partition, drop_sql:$drop_sql"
    if [[ -z "$partition" ]]; then
       echo "partitopn null, continue"
       continue
    fi
    clickhouse-client --host="$host" --password="$password" --query="$drop_sql"
  done <<< "$partitions"

done <<< "$hosts"
echo "done."
```
如上的脚本是删除指定数据库的指定表的过期partition,shell中的一些字段解释如下
- passsword: ck节点的密码，我的ck集群的所有分片/副本的密码是一致的
- cluster: ck集群的名称，通过集群名称查询集群下的所有分片和副本的host
- database: 要删除的数据库
- table: 要删除的具体的表
- end_date: 删除end_date之前的partition，我的表的partition格式是yyyyMMdd

首先查询system.clusters表，获得集群里的所有节点的host
然后遍历所有的host，挨个删除本地表上的过期的partition,小于end_date的partition全部删除

 
