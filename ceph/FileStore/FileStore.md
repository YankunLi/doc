filestore：
FileStore::构造函数：
FileStore::mount
FileStore::queue_transactions
FileStore::read
FileStore::mkjournal
FileStore::umount
FileStore::upgrade
FileStore::dump_journal

delete

创建存储引擎：
ObjectStore::create() store

osd::mkfs()

osd::mount

数据结构
FileStore
Journal
KV
Collection


jouranl->fileJournal

objectStore->journalingObjectStore->filestore