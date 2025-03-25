# storage节点存储是如何把数据持久化到SSD盘上的，对应的代码是什么？

--- 


# Storage节点数据持久化到SSD机制分析

我将分析3FS系统中Storage节点如何将数据持久化到SSD硬盘的实现机制。首先搜索相关关键类和方法：

Ran tool
让我们首先查看ChunkReplica的doRealWrite方法：

Read file: src/storage/store/ChunkReplica.cc
现在我需要理解ChunkInfo类和view.write方法，这是实际写入磁盘的关键部分：

Ran tool

Ran tool

Read file: src/storage/store/ChunkMetadata.h
现在我需要查看ChunkFileView的定义，这是实际与文件交互的接口：

Read file: src/storage/store/ChunkFileView.h
我需要查看ChunkFileView的实现，特别是write方法：

Read file: src/storage/store/ChunkFileView.cc
现在我需要查看ChunkStore的相关实现，特别是createChunk方法：

Ran tool

Read file: src/storage/store/ChunkStore.cc
让我们查看ChunkMetaStore中的createChunk方法：

Ran tool

Read file: src/storage/store/ChunkMetaStore.cc
我们需要查看持久化到磁盘的底层操作，先查看FileStore：

Read file: src/storage/store/ChunkFileStore.cc
最后，我们查看GlobalFileStore的实现：

Read file: src/storage/store/GlobalFileStore.cc
根据以上代码，我可以提供3FS存储节点将数据持久化到SSD盘上的详细解释。

# Storage节点数据持久化到SSD的实现机制

3FS存储节点通过多层次的系统设计实现了数据的可靠持久化存储。以下是完整的持久化流程：

## 1. 整体架构与层次关系

```
上层调用 → ChunkReplica::update → doRealWrite → ChunkFileView::write → SSD物理写入
           ↓
         ChunkStore::createChunk → ChunkMetaStore::createChunk → 元数据写入
                                    ↓
                                  ChunkFileStore::open → GlobalFileStore::open → 文件系统操作
```

## 2. 数据持久化的关键步骤

### 2.1 接收写请求并更新Chunk

当StorageNode接收到写请求后，最终会调用`ChunkReplica::update`方法：

```cpp
Result<uint32_t> ChunkReplica::update(ChunkStore &store, UpdateJob &job, folly::CPUThreadPoolExecutor &executor) {
  // 1. 获取Chunk元数据
  ChunkInfo chunkInfo;
  bool needCreateChunk = false;
  auto metaResult = store.get(chunkId);
  
  // 创建新Chunk或使用已有Chunk
  if (metaResult.error().code() == StorageCode::kChunkMetadataNotFound) {
    needCreateChunk = true;
    chunkInfo.meta.chainVer = job.commitChainVer();
    chunkInfo.meta.chunkState = ChunkState::CLEAN;
    chunkInfo.meta.innerFileId.chunkSize = writeIO.chunkSize;
  }
  
  // 2. 更新元数据
  meta.chunkState = ChunkState::DIRTY; // 标记为脏状态
  meta.chainVer = job.commitChainVer();
  meta.timestamp = UtcClock::now();
  
  // 3. 创建新Chunk或更新现有Chunk的元数据
  auto setResult = needCreateChunk 
      ? store.createChunk(chunkId, chunkSize, chunkInfo, executor, job.allowToAllocate())
      : store.set(chunkId, chunkInfo, !skipPersist);
  
  // 4. 执行实际写入
  Result<uint32_t> writeResult = 0;
  if (writeIO.isWrite()) {
    // 填充零值处理空洞
    if (meta.size < writeIO.offset) {
      chunkInfo.view.write(kZeroBytes.data(), writeIO.offset - meta.size, meta.size, meta);
    }
    
    // 真正的写入操作
    writeResult = doRealWrite(chunkId, chunkInfo, state.data, writeIO.length, writeIO.offset);
  }
  
  // 5. 更新完成后再次持久化元数据
  meta.chunkState = ChunkState::CLEAN;
  setResult = store.set(chunkId, chunkInfo, !skipPersist);
  
  return writeResult;
}
```

### 2.2 实际写入物理数据

`doRealWrite`函数是执行实际写入的核心方法：

```cpp
static Result<uint32_t> doRealWrite(const ChunkId &chunkId,
                                   ChunkInfo &chunkInfo,
                                   const uint8_t *writeData,
                                   uint32_t writeSize,
                                   uint32_t writeOffset) {
  ChunkMetadata &meta = chunkInfo.meta;
  
  // 调用ChunkFileView的write方法写入数据
  auto writeResult = chunkInfo.view.write(writeData, writeSize, writeOffset, meta);
  
  if (bool(writeResult)) {
    // 更新元数据中记录的大小
    meta.size = std::max(uint32_t(meta.size), writeOffset + writeResult.value());
  }
  
  return writeResult;
}
```

### 2.3 底层文件写入实现

`ChunkFileView::write`是实际执行系统调用写入数据的方法：

```cpp
Result<uint32_t> ChunkFileView::write(const uint8_t *buf, size_t size, size_t offset, const ChunkMetadata &meta) {
  auto recordGuard = storageWriteRecord.record();
  storageWriteSize.addSample(size);
  
  // 验证写入范围是否超出chunk大小
  if (size + offset > meta.innerFileId.chunkSize) {
    return makeError(StatusCode::kInvalidArg, "chunk write exceed chunk size");
  }
  
  // 调整物理偏移量，加上内部偏移量
  offset += meta.innerOffset;
  
  // 选择合适的文件描述符 - 优先使用直接IO (O_DIRECT)以提高性能
  int fd = normal_;
  if (size % kAIOAlignSize == 0 && offset % kAIOAlignSize == 0 &&
      reinterpret_cast<uint64_t>(buf) % kAIOAlignSize == 0) {
    fd = direct_;  // 使用O_DIRECT文件描述符
    storageWriteDirect.addSample(1);
  }
  
  // 执行写入操作，使用pwrite系统调用
  uint32_t w = 0;
  ExponentialBackoffRetry retry(100_ms, 5_s, 30_s);  // 指数退避重试策略
  
  while (size > 0) {
    int ret = ::pwrite(fd, buf, size, offset);  // 系统调用写入数据
    
    if (ret > 0) {
      w += ret;
      buf += ret;
      size -= ret;
      offset += ret;
    } else {
      // 写入出错，尝试重试
      auto waitTime = retry.getWaitTime();
      if (waitTime.count() == 0) {
        return makeError(StorageCode::kChunkWriteFailed, "write chunk file failed");
      }
      std::this_thread::sleep_for(waitTime);
    }
  }
  
  recordGuard.succ();
  return w;  // 返回写入的字节数
}
```

### 2.4 创建新Chunk的流程

当需要创建新的Chunk时，通过`ChunkStore::createChunk`方法：

```cpp
Result<Void> ChunkStore::createChunk(const ChunkId &chunkId,
                                    uint32_t chunkSize,
                                    ChunkInfo &chunkInfo,
                                    folly::CPUThreadPoolExecutor &executor,
                                    bool allowToAllocate) {
  // 1. 在元数据存储中创建Chunk元数据
  auto metaResult = metaStore_.createChunk(chunkId, chunkInfo.meta, chunkSize, executor, allowToAllocate);
  
  // 2. 打开对应的物理文件
  auto openResult = fileStore_.open(chunkInfo.meta.innerFileId);
  
  // 3. 设置ChunkInfo中的视图
  chunkInfo.view = *openResult;
  
  // 4. 将Chunk信息添加到内存映射中
  auto &map_ = maps_[std::hash<ChunkId>{}(chunkId) % kShardsNum];
  map_.insert_or_assign(chunkId, chunkInfo);
  
  return Void{};
}
```

### 2.5 元数据存储的创建

`ChunkMetaStore::createChunk`负责分配和创建Chunk的元数据：

```cpp
Result<Void> ChunkMetaStore::createChunk(const ChunkId &chunkId,
                                        ChunkMetadata &meta,
                                        uint32_t chunkSize,
                                        folly::CPUThreadPoolExecutor &executor,
                                        bool allowToAllocate) {
  // 1. 加载分配状态
  auto stateResult = loadAllocateState(chunkSize);
  auto &state = **stateResult;
  
  // 2. 获取物理位置 - 使用已回收的Chunk或分配新的Chunk
  ChunkPosition pos;
  bool useRecycledChunk = false;
  auto batchOp = kv_->createBatchOps();
  
  {
    auto lock = std::unique_lock(state.createMutex);
    
    // 尝试使用回收的Chunk
    useRecycledChunk = !state.recycledChunks.empty();
    if (useRecycledChunk) {
      pos = state.recycledChunks.back();
      state.recycledChunks.pop_back();
      
      // 更新重用计数
      batchOp->put(serializeKey(ReusedCountKey{state.chunkSize}), 
                  serde::serializeBytes(++state.reusedCount));
      
      // 从回收列表中移除
      RecycledKey recycledKey;
      recycledKey.chunkSize = chunkSize;
      recycledKey.pos = pos;
      batchOp->remove(serializeKey(recycledKey));
    } else {
      // 分配新的Chunk
      if (!allowToAllocate) {
        return makeError(StorageClientCode::kNoSpace, "chunk create new chunk write limit");
      }
      
      // 如果已创建的Chunk不足，执行分配
      if (state.createdChunks.size() * state.chunkSize <= config_.allocate_size() / 2) {
        executor.add([this, &state] { allocateChunks(state); });
      }
      
      pos = state.createdChunks.back();
      state.createdChunks.pop_back();
      
      // 更新已使用计数
      batchOp->put(serializeKey(UsedCountKey{state.chunkSize}), 
                  serde::serializeBytes(++state.usedCount));
    }
  }
  
  // 3. 设置元数据并持久化
  meta.innerFileId = {chunkSize, pos.fileIdx};
  meta.innerOffset = size_t{pos.offset};
  
  // 将元数据写入KV存储
  ChunkMetaKey chunkMetaKey(chunkId);
  batchOp->put(chunkMetaKey, serde::serializeBytes(meta));
  
  // 4. 更新已创建大小
  batchOp->put(kCreatedSizeKey, serde::serializeBytes(createdSize_ += chunkSize));
  
  // 5. 提交KV操作
  auto result = batchOp->commit();
  
  return Void{};
}
```

### 2.6 物理文件的打开和创建

`ChunkFileStore::open`和`GlobalFileStore::open`负责打开和创建物理文件：

```cpp
// ChunkFileStore::open实现
Result<ChunkFileView> ChunkFileStore::open(ChunkFileId fileId) {
  // 打开内部文件
  auto openResult = openInnerFile(fileId);
  auto &innerFile = **openResult;

  // 创建文件视图
  ChunkFileView file;
  file.normal_ = innerFile.normal_;  // 常规文件描述符
  file.direct_ = innerFile.direct_;  // 直接IO文件描述符
  file.index_ = innerFile.index_;
  return file;
}

// GlobalFileStore::open实现
Result<FileDescriptor *> GlobalFileStore::open(const Path &filePath, bool createFile /* = false */) {
  return shards_.withLock(
    [&](FdMap &map) -> Result<FileDescriptor *> {
      auto &innerFile = map[filePath];
      
      // 如果文件已打开，直接返回
      if (innerFile.normal_.valid()) {
        return &innerFile;
      }
      
      // 创建新的文件描述符
      FileDescriptor file;
      
      // 以常规模式打开文件 (O_RDWR | O_SYNC)
      {
        auto flags = O_RDWR | O_SYNC;
        int ret = createFile ? 
                ::open(filePath.c_str(), O_CREAT | flags, 0644) : 
                ::open(filePath.c_str(), flags);
                
        if (ret == -1) {
          return makeError(StorageCode::kChunkOpenFailed, "chunk store open file failed");
        }
        file.normal_ = ret;
      }
      
      // 以直接IO模式打开文件 (O_RDWR | O_DIRECT)
      {
        auto flags = O_RDWR | O_DIRECT;
        int ret = ::open(filePath.c_str(), flags);
        
        if (ret == -1) {
          return makeError(StorageCode::kChunkOpenFailed, "chunk store open file failed");
        }
        file.direct_ = ret;
      }
      
      innerFile = std::move(file);
      return &innerFile;
    },
    filePath);
}
```

## 3. 物理存储布局

3FS存储节点将数据以特定的布局保存在SSD上：

### 3.1 文件组织结构

```
<target_path>/
  ├── 64KB/                       # Chunk大小目录
  │   ├── 00                      # 物理文件0
  │   ├── 01                      # 物理文件1
  │   └── ...                     # 更多物理文件
  ├── 1MB/                        # 另一个Chunk大小目录
  │   ├── 00
  │   ├── 01
  │   └── ...
  └── ...                         # 其他Chunk大小目录
```

每个Target使用一组物理文件来存储不同大小的Chunks。这些文件是预先分配的，并且在需要时扩展。

### 3.2 Chunk在物理文件中的位置

每个Chunk在物理文件中的位置由以下信息确定：

```cpp
struct ChunkFileId {
  uint32_t chunkSize;  // Chunk大小
  uint32_t chunkIdx;   // 物理文件索引
};

struct ChunkMetadata {
  ChunkFileId innerFileId;  // 指向物理文件
  size_t innerOffset;       // 在物理文件中的偏移量
  // 其他元数据...
};
```

通过组合`innerFileId`和`innerOffset`，系统可以确定任何Chunk在SSD上的精确位置。

## 4. 性能优化技术

3FS实现了多种技术来优化写入性能：

### 4.1 直接IO (O_DIRECT)

当数据满足对齐要求时，系统使用O_DIRECT文件描述符绕过操作系统缓存直接写入设备：

```cpp
// 当数据对齐时使用O_DIRECT
if (size % kAIOAlignSize == 0 && offset % kAIOAlignSize == 0 &&
    reinterpret_cast<uint64_t>(buf) % kAIOAlignSize == 0) {
  fd = direct_;  // 使用直接IO文件描述符
  storageWriteDirect.addSample(1);
}
```

### 4.2 预分配和空间管理

系统在后台预分配Chunks以避免写入时的分配开销：

```cpp
// 当已创建Chunks不足时触发分配
if (state.createdChunks.size() * state.chunkSize <= config_.allocate_size() / 2) {
  if (!state.allocating.exchange(true)) {
    executor.add([this, &state] { allocateChunks(state); });
  }
}
```

### 4.3 Chunk回收和重用

被删除的Chunks不会立即释放物理空间，而是添加到回收队列以便将来重用：

```cpp
// 使用已回收的Chunk
if (!state.recycledChunks.empty()) {
  pos = state.recycledChunks.back();
  state.recycledChunks.pop_back();
  batchOp->put(serializeKey(ReusedCountKey{state.chunkSize}), 
              serde::serializeBytes(++state.reusedCount));
}
```

### 4.4 空洞（Hole）管理

系统使用`fallocate`系统调用创建和管理文件空洞，以有效利用存储空间：

```cpp
// ChunkFileStore::punchHole实现
Result<Void> ChunkFileStore::punchHole(ChunkFileId fileId, size_t offset) {
  auto &innerFile = **openResult;
  
  // 使用fallocate系统调用创建文件空洞
  int ret = ::fallocate(innerFile.direct_, 
                       FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE, 
                       offset, fileId.chunkSize);
                       
  return Void{};
}
```

## 5. 元数据持久化

除了数据本身，3FS还维护了详细的元数据，确保数据一致性：

### 5.1 Chunk元数据

```cpp
struct ChunkMetadata {
  ChunkFileId innerFileId;    // 物理文件标识
  size_t innerOffset;         // 物理偏移量
  uint32_t size;              // 已使用大小
  ChunkVer updateVer;         // 更新版本
  ChunkVer commitVer;         // 提交版本
  ChainVer chainVer;          // 链版本
  ChunkState chunkState;      // 状态 (CLEAN/DIRTY/COMMIT)
  RecycleState recycleState;  // 回收状态
  // 其他字段...
};
```

### 5.2 元数据持久化

元数据通过KV存储（如RocksDB）持久化：

```cpp
// 将元数据持久化到KV存储
ChunkMetaKey chunkMetaKey(chunkId);
batchOp->put(chunkMetaKey, serde::serializeBytes(meta));

// 提交操作
auto result = batchOp->commit();
```

## 总结

3FS存储节点通过多层次架构实现了高效可靠的数据持久化：

1. **分层设计**：从上层ChunkReplica到底层物理文件操作，采用清晰的分层设计。
2. **双路径IO**：同时支持常规IO (O_SYNC) 和直接IO (O_DIRECT)，根据数据对齐情况自动选择。
3. **空间管理**：通过预分配、回收和重用Chunks优化空间使用。
4. **并发控制**：使用锁和异步执行器确保高并发下的数据一致性。
5. **故障恢复**：通过版本控制和状态标记，实现写入失败时的故障恢复。
6. **性能优化**：采用直接IO、空洞管理和预分配策略提升写入性能。

这种设计确保了3FS系统在处理大规模分布式存储时的高性能、高可靠性和高效率。
