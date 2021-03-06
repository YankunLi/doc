# RocksDB VersionSet

VersionSet中维护一组ColumnFamilyData集合,每个ColumnFamilyData中维护一个Version双向链表.ColumnFamilyData::current_指向最新的version实例.

## 方法介绍

Version VersionSet ColumnFamilyData

### VersionSet

VersionSet中包含ColumnFamilyData的集合(ColumnFamilySet),每个ColumnFamilyData中有自己的Version集合,每个version中又维护了每一层的文件描述对象.

* VersionSet::AppendVersion(ColumnFamilyData* column_family_data,Version* v)
将Version v插入到ColumnFamilyData维护的Version环形双向链表的表头,并将ColumnFamilyData::current_指向该version v,老的 current_ version引用计数减一,新的current_ version加一,即更新ColumnFamilyData中当前version;

* Status VersionSet::LogAndApply(const std::vector<ColumnFamilyData*>& column_family_datas,const std::vector\<MutableCFOptions\>& mutable_cf_options_list, std::vector<autovector<VersionEdit*>>& edit_lists,InstrumentedMutex* mu, Directory* db_directory, bool new_descriptor_log,const ColumnFamilyOptions* new_cf_options)
先做传入的参数合法性检查,将ColumnFamilyData和VersionEdit集合,一组ManifestWriter对象,再调用ProcessManifestWrites处理ManifestWriter集合;

* void VersionSet::LogAndApplyHelper(ColumnFamilyData* cfd,VersionBuilder* builder, Version* /*v*/,VersionEdit* edit, InstrumentedMutex* mu)
将

* ColumnFamilyData* VersionSet::CreateColumnFamily(const ColumnFamilyOptions& cf_options, VersionEdit* edit)
创建新的ColumnFamilyData,并创建一个Version保存到被创建的ColumnFamilyData中.

* void VersionSet::AddLiveFiles(std::vector<FileDescriptor>* live_list)
将VersionSet中的所有ColumnFamilyData中的所有version中的每一层文件描述对象,添加到live_list集合中.

* Status VersionSet::GetMetadataForFile(uint64_t number, int* filelevel,FileMetaData** meta,ColumnFamilyData** cfd)
遍历VersionSet中每个ColumnFamilyData的current version每一层,查找编号为number的文件,并获取该文件所在ColumnFamilyData,原信息FileMetaData和所在层级.

* void VersionSet::GetLiveFilesMetaData(std::vector<LiveFileMetaData>* metadata)
遍历VersionSet中的所有ColumnFamilyData的每个Current version每一层的所有文件,然后将文件的原信息保存封装在LiveFileMetaData对象中,并保存到metadata中返回.

* void VersionSet::MarkFileNumberUsed(uint64_t number)
将number中VersionSet中标记已经被使用,标记方法即使VersionSet::next_file_number_的数值比number大1.

* void VersionSet::MarkMinLogNumberToKeep2PC(uint64_t number)
经VersionSet::min_log_number_to_keep_2pc_更新为number.

* void VersionSet::GetObsoleteFiles(std::vector<ObsoleteFileInfo>* files, std::vector<std::string>* manifest_filenames,uint64_t min_pending_output)
获取VersionSet中维护的废弃的文件且这些文件的编号小于min_pending_output,将这些废弃的文件信息保存到files中返回,同时页获取VersionSet中废弃的
manifest文件信息.

uint64_t VersionSet::GetNumLiveVersions(Version* dummy_versions)
dummy_versions是version双向链表的表头,该函数统计该链中现存的version数量,并返回.

* uint64_t VersionSet::GetTotalSstFilesSize(Version* dummy_versions)
统计dummy_versions version链表中所有version的每个level中的每个文件大小之后,且文件不重复统计(即同一个文件只统计一次),并返回该version链中包含的文件大小总和.

* Status VersionSet::LogAndApply(const std::vector<ColumnFamilyData*>& column_family_datas, const std::vector<MutableCFOptions>& mutable_cf_options_list, const std::vector<autovector<VersionEdit*>>& edit_lists, InstrumentedMutex* mu, Directory* db_directory, bool new_descriptor_log,const ColumnFamilyOptions* new_cf_options)
将集合column_family_datas,mutable_cf_options_list和edit_lists中的元素按其顺序组合创建ManifestWriter实例,并保存到临时集合writes(std::deque<ManifestWriter>)中,同时将保存writes中所有元素的地址保存到VersionSet::manifest_writers_属性中,然后调用ProcessManifestWrites,writes作为其中的参数.

* void VersionSet::LogAndApplyCFHelper(VersionEdit* edit)
为VersionEdit记录设置next_file_name_,如果该VersionEdit是删除Column Family(is_column_family_drop_为true) 需要将最大的ColumnFamily id赋值给max_column_family_; 更新VersionEdit中所属VersionSet的last_allocated_sequence_,last_published_sequence_,last_sequence_这三个属性相同;

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
将VersionBuilder::Rep中维护的当前状态,和上一个版本的Version一同合并(保存到)新的Version(VersionStoreageInfo)中.

* void MaybeAddFile(VersionStorageInfo* vstorage, int level, FileMetaData* f)
根据VersionBuilder::Rep中维护的状态,来处理level层的文件f,如果文件f是在被删除的集合中,就将vstorage中的该文件删除,如果该文件f在新增集合中;

* void UnrefFile(FileMetaData* f)
减少文件对象f的引用计数,如果其引用计数小于等于0,就删除其内存对象;

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
