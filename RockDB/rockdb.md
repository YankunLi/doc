# RockDB

PUT:

```c++
virtual Status Put(const WriteOptions& options, const Slice& key,const Slice& value)   //include/rockdb/db.h
```

db/db_impl_write.cc

```c++
Status DBImpl::Put(const WriteOptions& o, ColumnFamilyHandle* column_family,
                   const Slice& key, const Slice& val) {
  return DB::Put(o, column_family, key, val);
}
```

db/db_impl_write.cc

```c++
// Default implementations of convenience methods that subclasses of DB
// can call if they wish
Status DB::Put(const WriteOptions& opt, ColumnFamilyHandle* column_family,
               const Slice& key, const Slice& value) {
  // Pre-allocate size of write batch conservatively.
  // 8 bytes are taken by header, 4 bytes for count, 1 byte for type,
  // and we allocate 11 extra bytes for key length, as well as value length.
  WriteBatch batch(key.size() + value.size() + 24);
  Status s = batch.Put(column_family, key, value);
  if (!s.ok()) {
    return s;
  }
  return Write(opt, &batch);
}
```

将外部传入的参数转换成内部操作对象WriteBatch;
WriteBatch会预申请一些空间,除了用于保存KV的空间外,需要24Byte的空间:8byte头信息,4byte计数信息,1byte的类型信息,11byte的扩展空间用于key长度表示

## WriteBatch TODO

WriteBatch继承WriteBatchBase;

db/write_batch.cc

```c++
Status WriteBatch::Put(ColumnFamilyHandle* column_family, const Slice& key, 
                       const Slice& value) {
  return WriteBatchInternal::Put(this, GetColumnFamilyID(column_family), key, 
                                 value);
}
```

db/write_batch.cc

```c++
Status WriteBatchInternal::Put(WriteBatch* b, uint32_t column_family_id,
                               const Slice& key, const Slice& value) {
  if (key.size() > size_t{port::kMaxUint32}) {
    return Status::InvalidArgument("key is too large");
  }
  if (value.size() > size_t{port::kMaxUint32}) {
    return Status::InvalidArgument("value is too large");
  }

  LocalSavePoint save(b);
  WriteBatchInternal::SetCount(b, WriteBatchInternal::Count(b) + 1);
  if (column_family_id == 0) {
    b->rep_.push_back(static_cast<char>(kTypeValue));
  } else {
    b->rep_.push_back(static_cast<char>(kTypeColumnFamilyValue));
    PutVarint32(&b->rep_, column_family_id);
  }
  PutLengthPrefixedSlice(&b->rep_, key);
  PutLengthPrefixedSlice(&b->rep_, value);
  b->content_flags_.store(
      b->content_flags_.load(std::memory_order_relaxed) | ContentFlags::HAS_PUT,
      std::memory_order_relaxed);
  return save.commit();
}
```

db/db_impl_write.cc

```c++
Status DBImpl::Write(const WriteOptions& write_options, WriteBatch* my_batch) {
  return WriteImpl(write_options, my_batch, nullptr, nullptr);
}
```

```c++
将写操作传入的参数封装成WriteBatch对象,继续执行后续写入操作;

  // If disable_memtable is set the application logic must guarantee that the
  // batch will still be skipped from memtable during the recovery. An excption
  // to this is seq_per_batch_ mode, in which since each batch already takes one
  // seq, it is ok for the batch to write to memtable during recovery as long as
  // it only takes one sequence number: i.e., no duplicate keys.
  // In WriteCommitted it is guarnateed since disable_memtable is used for
  // prepare batch which will be written to memtable later during the commit,
  // and in WritePrepared it is guaranteed since it will be used only for WAL
  // markers which will never be written to memtable. If the commit marker is
  // accompanied with CommitTimeWriteBatch that is not written to memtable as
  // long as it has no duplicate keys, it does not violate the one-seq-per-batch
  // policy.
  // batch_cnt is expected to be non-zero in seq_per_batch mode and
  // indicates the number of sub-patches. A sub-patch is a subset of the write
  // batch that does not have duplicate keys.
  Status WriteImpl(const WriteOptions& options, WriteBatch* updates,
                   WriteCallback* callback = nullptr,
                   uint64_t* log_used = nullptr, uint64_t log_ref = 0,
                   bool disable_memtable = false, uint64_t* seq_used = nullptr,
                   size_t batch_cnt = 0,
                   PreReleaseCallback* pre_release_callback = nullptr);
```
