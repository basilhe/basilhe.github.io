---
layout: post
title: "Large Scale Data Storage with MongoDB Cluster or Cassandra"
description: ""
category: 
tags: [sharding]
---
{% include JB/setup %}

# 目标

* 解决设备上报和控制指令并发读写存储问题
* 解决日益增长的指令存储空间问题

# 思路

* 采用支持读写高并发的数据库解决读写并发问题
* 采用分布式数据库解决空间增长问题
* 采用 Cassandra 分布式数据库

# 分析

## RabbitMQ 日志消息 JSON 格式
	{    
	type="dev_reg"|"user_reg"|"dev_online"|"dev_offline"|"dev_re_online"|"user_online"
	  |"user_offline"|"dev_ping"|"app2dev"|"dev2app"|"dev_protocol_ver", 
    did=<str>,
    uid=<str>,
    mac=<str>,
    product_key=<str>,
    timestamp=<float in seconds>, 
    ip=<str>,
    gen_protocol=<str>,
    busi_protocol=<base64_str>,
    payload=base64(<payloadBin>) or json string(if type=online|offline)
	}

* "dev_reg" requires the following fields: did, mac, product_key, timestamp, ip
* "user_reg" requires the following fields: appid, uid, timestamp, ip
* "dev_online" requires the following fields: did, mac(in some cases, the value would be ""), product_key, timestamp, ip, payload(string, '{"keepalive: <int-sec>}')
* "dev_offline" requires the following fields: did, mac(in some cases, the value would be ""), product_key, timestamp, ip, payload(string, '{"reason: "str", "duration": <int-sec>, * "heartbeat": {"max": <int-sec>, "min": <int-sec>, "avg": <int-sec>, "count": <int>, "last": <int>}}')
* "dev_re_online" requires the following fields: did, mac(in some cases, the value would be ""), product_key, timestamp, ip, payload(string, '{"reason: "str", "duration": <int-sec>, "heartbeat": {"max": <int-sec>, "min": <int-sec>, "avg": <int-sec>, "count": <int>, "last": <int>}}')* 
* "user_online" requires the following fields: appid, uid, timestamp, ip, payload(string, '{"keepalive: <int-sec>}')
* "user_offline" requires the following fields: appid, uid, timestamp, ip, payload(string, '{"reason: "str", "duration": <int-sec>, "heartbeat": {"max": <int-sec>, "min": <int-sec>, * "avg": <int-sec>, "count": <int>, "last": <int>}}')
* "dev_ping" requires the following fields: did, product_key, timestamp
* **"app2dev" requires the following fields: appid, uid, did, mac, product_key, timestamp, ip, payload**
* **"dev2app" requires the following fields: did, mac, product_key, timestamp, ip, payload**
* "dev_protocol_ver" requires the following fields: did, mac, product_key, timestamp, gen_protocol, busi_protocol


## 日志数据量查询

目前最大的数据量为 dev2app，其次是 app2dev，这两类数据结构很接近

查询主要参数为 DID 和 时间范围，排序方式为时间倒序，即最近的数据优先，需要进行翻页查询

# 方案 I

## Cassandra 模型

Cassandra 是面向查询建模的，根据查询关键属性 type(主要区分dev2app和app2dev，可再分表去除该属性)、DID和时间，采用 type 和 DID 作为 RowKey，时间 when 作为 Clustering Column 并使用降序排序

每个 RowKey 可以存数 20 亿条数据，数据需要在单台机器上能够全部存储。

### 设备上报和下发控制指令

	CREATE KEYSPACE raw WITH replication
	  = {'class':'SimpleStrategy', 'replication_factor':3}; 
	  
	USE raw;
	
	DROP TABLE ods_cmd;
  	CREATE TABLE ods_cmd(
    	type text, -- 'dev2app' | 'app2dev'
    	did text,
	    mac text,
    	prdkey text,
	    when bigint,
	    date int, -- 15091023， 2015-09-10 23:00:00 时间
    	ip inet,
	    data text, -- payload
    	appid text,
	    uid text,
    	PRIMARY KEY((type,did), when) -- RowKey， Cluster Column
	) WITH CLUSTERING ORDER BY (when DESC); -- 按时间倒序排序
	CREATE INDEX ON ods_cmd(date); -- 按小时索引数据，方便数据导出
  		
	INSERT INTO ods_cmd (type, did, mac, prdkey, when, ip, data, appid, uid)
	  VALUES('dev2app', 'YvsLoJxhsZYGNmQiREVYpK', 'mac', 'f450adc5af54492a9ac639845add7494', 1441900898000, null, 'AAAAAxcAAJMAAAAAAwAQAAAAFAACAAAAAAsAAA==','app01', 'uid01');
	  
	  # 分页
	  select * from ods_cmd where type='dev2app' and did='YvsLoJxhsZYGNmQiREVYpK' and when > 1441900798000 limit 10;
	  
### 数据有效时间 TTL 

可以在插入数据时指定 TTL 

	INSERT INTO ods_cmd (type, did, mac, prdkey, when, ip, data, appid, uid)
  		VALUES('dev2app', 'YvsLoJxhsZYGNmQiREVYpK', 'mac', 'f450adc5af54492a9ac639845add7494', 1441900798000, '10.10.1.5', 'AAAAAxcAAJMAAAAAAwAQAAAAFAACAAAAAAsAAA==', 'app01', 'uid01') USING TTL 864000;
	
### 数据导入和导出

#### 数据量较小时使用 COPY 指令

	COPY ods_cmd (type, did, prdkey, when, data, date) FROM 'ods_cmd.csv';

COPY FROM is intended for importing small datasets (a few million rows or less) into Cassandra. For importing larger datasets, use the [Cassandra bulk loader](http://docs.datastax.com/en/cassandra/2.0/cassandra/tools/toolsBulkloader_t.html).

#### 大量数据导入使用 sstableloader
https://github.com/yukim/cassandra-bulkload-example/blob/master/src/main/java/bulkload/BulkLoad.java

#### 数据导出
https://github.com/gianlucaborello/cassandradump
https://moshimon.wordpress.com/2015/01/19/export-data-from-cassandra-to-csv/

## 移植限制

	select * from ods_cmd where type='dev2app' and did='YvsLoJxhsZYGNmQiREVYpK' and when > 1441900798000 limit 10;
查询条件必须输入 type、did，时间范围可选，通过指定 when 和 limit 组合进行分页，由于数据量可能巨大，获取总数可能会很慢，建议取消获取分页总数

## Python 驱动

	$ pip install cassandra-driver 
	
## Python 代码样例
	
	from cassandra.cluster import Cluster
	from cassandra.query import BatchStatement
	from cassandra import ConsistencyLevel
	
	cluster = Cluster(['127.0.0.1'])
	session = cluster.connect()
	
	results = session.execute("select * from raw.ods_cmd limit 5")
	for row in results:
	  print"%-20s\t%-20s\t%-20s" % (row.did, row.prdkey, row.when)
	
	prepared_statement = session.prepare("""
		INSERT INTO raw.ods_cmd 
		  (type, did, prdkey, when, data, date) 
		VALUES (?, ?, ?, ?, ?, ?);
	""")
	session.execute(prepared_statement, 
		['dev2app', 'did1', 'pk1', 1441900798000, 'AAB==', '15091023']
	)
	 
	// Batch Insert  ConsistencyLevel.QUORUM
	batch = BatchStatement(consistency_level=ConsistencyLevel.ONE)

	for i in range(0, 10000):
  	  batch = BatchStatement(consistency_level=ConsistencyLevel.QUORUM)
      for y in range(100, 109):
        for z in range(1,5):
          batch.add(prepared_statement, ('dev2app', 'did%d' % z, 'pk1', 1441900798000+i+y, 'AQ=', 150910))
      session.execute(batch)
      
### 注意控制 BATCH 大小，否则写入速度会明显下降
默认 BATCH 大小限制为 5kb 

### 其他：数据冗余和备份、恢复

可以考虑 Kafka 进行冗余存储，需要寻求或定制数据的备份和快速恢复方案

## References

* [Gizwits M2M Data API](http://redmine.xtremeprog.com/projects/xcloud/wiki/Gizwits_m2m_data_api)
* [Import CSV Data Into Cassandra](http://docs.datastax.com/en/cql/3.1/cql/cql_reference/copy_r.html)
* [Python Driver](http://docs.datastax.com/en/developer/python-driver/2.7/index.html)
* [Python Driver Reference](http://docs.datastax.com/en/drivers/python/2.6/)

# 方案 II

## MongoDB Sharding

### 重点

* 使用 MongoDB Sharding
* 采用合适的 Sharding Keys
* 使用 TTL 自动过期 Collection 数据

### 步骤

#### 搭建 Sharding Cluster

* [MongoDB Cloud Manager](http://cloud.mongodb.com)
* [MongoDB Ops Manager](https://www.mongodb.com/products/ops-manager)

	
#### 创建查询索引

	db.gizwits_raw.createIndex({"did":1, "type":1, "timestamp":-1});

#### 创建 TTL 索引

由于 TTL 索引必须使用 ISODate，不能使用目前的 timestamp，为了避免代码改动，暂定增加新字段和索引

	db.gizwits_raw.createIndex( { "createdAt": 1 }, { expireAfterSeconds: 604800 } )

#### Enable Sharding for a Database

	sh.enableSharding("<database>")
	
	db.runCommand( { enableSharding: <database> } )
	
	use db.raw
	sh.enableSharding("raw")
	
#### Shard a collection

##### 选择 Sharding Key

* Command

		sh.shardCollection("<database>.<collection>", shard-key-pattern)
	
* Examples

		sh.shardCollection("records.people", { "zipcode": 1, "name": 1 } )
		sh.shardCollection("people.addresses", { "state": 1, "_id": 1 } )
		sh.shardCollection("assets.chairs", { "type": 1, "_id": 1 } )
		sh.shardCollection("events.alerts", { "_id": "hashed" } )

目前的查询主要基于 DID，而 DID 的量足够大，便于分散读写压力，所以初步采用 DID 作为 Sharding Key

	sh.shardCollection("raw.gizwits_raw", {"did": 1})

	db.gizwits_raw.insert({"did":"123", "type":"dev_online", "timestamp":1442908844.899, "createAt":new Date()})
	db.aos_raw.insert({"mac":"123", "type":"status", "timestamp":1442908844.899, "createAt":new Date()})
	
	db.gizwits_raw.createIndex({"did":1, "type":1, "timestamp":-1});
	db.aos_raw.createIndex({"mac":1, "type":1, "timestamp":-1});
	
	db.gizwits_raw.createIndex({"createdAt":1}, {expireAfterSeconds:604800})
	db.aos_raw.createIndex({"createdAt":1}, {expireAfterSeconds:604800})
	
	sh.enableSharding("raw")
	
	sh.shardCollection("raw.gizwits_raw", {"did": 1})

# MongoDB Sharding Cluster


## master,slave,facter三台机器上都做如下操作：

	mkdir -p /data/sharding/{mongos,config,shard1,shard2,shard3}
	cd /data/sharding
	mkdir -p mongos/{data,logs}
	mkdir -p config/{data,logs}
	mkdir -p shard1/{data,logs}
	mkdir -p shard2/{data,logs}
	mkdir -p shard3/{data,logs}

## master,slave,facter三台机器上都做如下操作（启动config server）：

	/data/mongodb/bin/mongod --configsvr --dbpath /data/sharding/config/data --port 26000 --logpath=/data/sharding/config/logs/config.log --logappend --fork

## master,slave,facter三台机器上都做如下操作（启动mongos路由）：

	/data/mongodb/bin/mongos --configdb 201.132.63.17:26000,201.121.216.53:26000,201.132.55.89:26000 --port 25000 --logpath=/data/sharding/mongos/logs/mongos.log --fork

## master,slave,facter三台机器上都做如下操作（启动mongod节点）：

	/data/mongodb/bin/mongod --shardsvr --replSet shard1 --port 26001 --dbpath=/data/sharding/shard1/data --logpath=/data/sharding/shard1/logs/shard1.log --fork --oplogSize 100 --logappend

## master,slave,facter三台机器上都做如下操作（启动mongod节点）：

	/data/mongodb/bin/mongod --shardsvr --replSet shard2 --port 26002 --dbpath=/data/sharding/shard2/data --logpath=/data/sharding/shard2/logs/shard2.log --fork --oplogSize 100 --logappend

## master,slave,facter三台机器上都做如下操作（启动mongod节点）：

	/data/mongodb/bin/mongod --shardsvr --replSet shard3 --port 26003 --dbpath=/data/sharding/shard3/data --logpath=/data/sharding/shard3/logs/shard3.log --fork --oplogSize 100 --logappend

## 在非[arbiterOnly:true]机器上都做如下操作（配置副本集shard1）：

	./mongo --port 26001
	use admin
	config={_id:"shard1",members:[{_id:0,host:"201.132.63.17:26001",arbiterOnly:true},{_id:1,host:"201.121.216.53:26001"},{_id:2,host:"201.132.55.89:26001"}]}
	rs.initiate(config);

## 在非[arbiterOnly:true]机器上都做如下操作（配置副本集shard2）：

	./mongo --port 26002
	use admin
	config={_id:"shard2",members:[{_id:0,host:"201.132.63.17:26002"},{_id:1,host:"201.121.216.53:26002",arbiterOnly:true},{_id:2,host:"201.132.55.89:26002"}]}
	rs.initiate(config);

## 在非[arbiterOnly:true]机器上都做如下操作（配置副本集shard3）：
	./mongo --port 26003
	use admin
	config={_id:"shard3",members:[{_id:0,host:"201.132.63.17:26003"},{_id:1,host:"201.121.216.53:26003"},{_id:2,host:"201.132.55.89:26003",arbiterOnly:true}]}
	rs.initiate(config);

## 在任意一台机器上都做如下操作（配置sharding）：
	./mongo --port 25000
	use admin
	db.runCommand({addshard:"shard1/201.132.63.17:26001,201.121.216.53:26001,201.132.55.89:26001"})
	db.runCommand({addshard:"shard2/201.132.63.17:26002,201.121.216.53:26002,201.132.55.89:26002"})
	db.runCommand({addshard:"shard3/201.132.63.17:26003,201.121.216.53:26003,201.132.55.89:26003"})

## 查看分片情况：
	db.runCommand({listshards:1})

## 设置需要分片的库表和分片方式：
	sh.enableSharding("raw")
	sh.shardCollection("raw.gizwits_raw", {"did":"1"})

## 后续

注意规划好副本集角色在机器上的分布以及根据实际情况调整分片的参数，如：oplogSize，chunkSize，balance的时间等参数。


# References

* [Expire Data](http://docs.mongodb.org/manual/tutorial/expire-data/)
* [搭建高可用 MongoDB 集群](http://www.lanceyan.com/tech/arch/mongodb_shard1.html)
