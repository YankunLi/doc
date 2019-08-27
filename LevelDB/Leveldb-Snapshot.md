# LevelDB 快照实现原理

## LevelDB Snapshot

LevelDB Snapshot是LevelDB某一时刻,数据库中的数据状态,即数据库创建快照时的一个映像,可以理解为数据库那一世界的一个副本.
LevelDB Snpashot 对这整个KV存储的状态,Snapshot提供了一致性只读视图.在读取操作时通过指定快照来读取某个快照中的内容.

### LevelDB创建快照

* DB::GetSnapshot()创建快照

  * 直接创建snapshot`:`

    ```c++
    levelDB::ReadOptions options;
    options.snapshot = db->GetSnapshot()
    ...apply some updates to db...
    leveldb::Iterator* iter = db->NewIterator(options);
    ... read using iter to view the state when the snapshot was created...
    delete iter;
    db->ReleaseSnapshot(options.snapshot);
    ```

  * 写操作也可以返回一个更新后的snapshot`:`

    ```C++
    levelDB::Snapshot* Snapshot;
    leveldb::WriteOptions write_options;
    write_options.post_write_snapshot = &snapshot; //更新后生产快照对象
    levedb::Status status = db->Write(write_options,...);
    ...perform other mutations to db...
    leveldb::ReadOptions read_options;
    read_options.snapshot = snapshot;
    leveldb::Iterator* iter = db->NewIterator(read_options);
    ...read as of the state just after the write call returned...
    delete iter;
    db->ReleaseSnapshot(snapshot);
    ```

* DB::ReleaseSnapshot释放快照.
Snapshot不再使用时,调用DB::ReleaseSnapshot释放快照,这样显示调用该接口,可以释放支持snapshot的资源.

### Snapshot实现原理

&emsp;&emsp;LevelDB在一段时间内只增加数据,并不删除数据,其操作实例如下:

   ```shell
   table["xiaomi"] = 20
   del table["xiaomi"]
   table["huawei"] = 30
   table["huawei"] = 31
   ```

&emsp;&emsp;这些内容在leveldb中的内部操作如下:

   ```shell
   xiaomi 1 kTypeValue : 20
   xiaomi 2 kTypeDeletion
   huawei 3 kTypeValue :30
   huawei 4 kTypeValue :31
   ```
　　
&emsp;&emsp;内部按照key非递减,sequence(第二列)非递增,kTypeValue非递增排序(保证kTypeValue在前面)进行排序(存储在SkipList中),在SkipList中的形式为:

   ```shell
   //key sequence type :value
   xiaomi 2 kTypeDeletion
   xiaomi 1 kTypeValue :20
   huawei 4 kTypeValue :30
   huawei 3 kTypeValue :31
   ```
  
&emsp;&emsp;客户端获取leveldb的快照时,会得到一个确的sequence号,即这个sequence可以理解位这个快照的代号(snapshot sequence),客户端读取时之后读取<=snapshot sequence的KB值,所以无论创建快照之后数据怎么变动都不会映像快照中的内容读取,保证的leveldb快照一致性视图特性.实现这个特性的前置条件:1.leveldb的更新操作,在一段时间内只增加不删除;2.每个更新操作都被编排一个一直递增的序号.

### LevelDB Not Persistence

&emsp;&emsp;LevelDB的快照不会在DB重启后仍然存在,即LevelDB的snapshot不支持持久化.
