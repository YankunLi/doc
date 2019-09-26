# RocksDB VersionSet

VersionSet中维护一组ColumnFamilyData集合,每个ColumnFamilyData中维护一个Version双向链表.ColumnFamilyData::current_指向最新的version实例.

## 方法介绍

Version VersionSet ColumnFamilyData

### VersionSet

* VersionSet::AppendVersion(ColumnFamilyData* column_family_data,Version* v)
将Version v插入到ColumnFamilyData维护的Version链表的表头,并将ColumnFamilyData::current_指向该version v,老的 current_ version引用计数减一,新的current_ version加一;

* Status VersionSet::LogAndApply(const std::vector<ColumnFamilyData*>& column_family_datas,const std::vector\<MutableCFOptions\>& mutable_cf_options_list, std::vector<autovector<VersionEdit*>>& edit_lists,InstrumentedMutex* mu, Directory* db_directory, bool new_descriptor_log,const ColumnFamilyOptions* new_cf_options)
先做传入的参数合法性检查,将ColumnFamilyData和VersionEdit集合,一组ManifestWriter对象,再调用ProcessManifestWrites处理ManifestWriter集合;

*void VersionSet::LogAndApplyHelper(ColumnFamilyData* cfd,VersionBuilder* builder, Version* /*v*/,VersionEdit* edit, InstrumentedMutex* mu)
将

### Version

* Version::Ref()
Version::UnRef()
对自身引用计数做加减一操作,如果引用计数为0,就删除自身对象;

### BaseReferencedVersionBuilder

BaseReferencedVersionBuilder封装了VersionBuilder类型,并引用了Version类型,主要用于构建新的version;

```c++
// A wrapper of version builder which references the current version in
// constructor and unref it in the destructor.
// Both of the constructor and destructor need to be called inside DB Mutex.
class BaseReferencedVersionBuilder {
 public:
  explicit BaseReferencedVersionBuilder(ColumnFamilyData* cfd)
      : version_builder_(new VersionBuilder(
            cfd->current()->version_set()->env_options(), cfd->table_cache(),
            cfd->current()->storage_info(), cfd->ioptions()->info_log)),
        version_(cfd->current()) {
    version_->Ref();
  }
  ~BaseReferencedVersionBuilder() {
    delete version_builder_;
    version_->Unref();
  }
  VersionBuilder* version_builder() { return version_builder_; }

 private:
  VersionBuilder* version_builder_;
  Version* version_;
};
```

### VersionBuilder

VersionBuilder的所有功能的所有具体实现都是由VersionBuilder::Rep完成,所以只需要了解VersionBuilder::Rep的具体实现.

### VersionBuilder::Rep

* void VersionBuilder::Rep::Apply(VersionEdit* edit)
将VersionEdit版本间的差异,应用到VersionBuilder(Rep)中维护的当前状态中;

* void SaveTo(VersionStorageInfo* vstorage)
将VersionBuilder::Rep中维护的当前状态,和上一个版本的Version一同合并新的Version(VersionStoreageInfo)中.

### VersionEdit

VersionEdit记录着两个版本间的差异(被删除或者被新添加的文件),在上一个版本的基础上应用VersionEdit就可以得到一个新的Version.
其中主要的两个属性:
    - deleted_files_(DeletedFileSet)                           //被删除的文件列表
    - new_files_(std::vector<std::pair<int, FileMetaData>>)    //新添加的文件列表

### FileMetaData

### VersionStorageInfo

### LevelStat

LevelStat内存中的对象,描述rocksdb中每一层删除和增加的文件.
    - std::unordered_set<uint64_t> deleted_files; //某层删除的文件
    - std::unordered_map<uint64_t, FileMetaData*> added_files; //某层新增的文件
