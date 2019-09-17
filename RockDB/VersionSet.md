# RocksDB VersionSet

VersionSet中维护一组ColumnFamilyData集合,每个ColumnFamilyData中维护一个Version双向链表.ColumnFamilyData::current_指向最新的version实例.

## 方法介绍

Version VersionSet ColumnFamilyData

### VersionSet

* VersionSet::AppendVersion(ColumnFamilyData* column_family_data,Version* v)
将Version v插入到ColumnFamilyData维护的Version链表的表头,并将ColumnFamilyData::current_指向该version v,老的 current_ version引用计数减一,新的current_ version加一;

### Version

* Version::Ref()
Version::UnRef()
对自身引用计数做加减一操作,如果引用计数为0,就删除自身对象;
