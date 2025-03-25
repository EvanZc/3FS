# 存储节点处理写IO与链式复制实现详解

---


# 存储节点处理写IO与链式复制实现详解

## 1. 存储节点接收写请求

当存储节点接收到来自客户端的写请求时，首先由`StorageServer`类处理：

Ran tool
让我查找存储服务的实现位置：

Ran tool

Read file: /d:/code/3FS/src/storage/service/StorageServer.h

Read file: src/storage/service/StorageServer.h
现在我需要查找StorageOperator的实现，它可能包含处理写请求的核心逻辑：

Read file: src/storage/service/StorageOperator.h
我看到在StorageOperator中有一个`update`方法和一个`write`方法，这应该是处理写请求的关键方法。让我查看这些方法的实现：

Ran tool

Read file: src/storage/service/StorageOperator.cc

Ran tool
继续阅读handleUpdate方法的剩余部分：

Read file: src/storage/service/StorageOperator.cc
让我查看更多细节，特别是关于UpdateWorker和UpdateJob的实现：

Ran tool

Read file: src/storage/update/UpdateJob.h
接下来，我们来看UpdateWorker的实现：

Read file: src/storage/update/UpdateWorker.h

Read file: src/storage/update/UpdateWorker.cc
现在我们来看一下Storage Target如何处理更新操作：

Ran tool

Read file: src/storage/store/StorageTarget.h

Ran tool

Read file: src/storage/store/StorageTarget.cc
我们需要看一下ChunkReplica和ChunkEngine中的update方法：

Ran tool

Read file: src/storage/store/ChunkReplica.cc

Read file: src/storage/store/ChunkReplica.cc
让我们看一下ChunkEngine的update方法：

Ran tool

Read file: src/storage/store/ChunkEngine.cc
现在，让我们探索一下ReliableForwarding类，它处理链式复制过程：

Read file: src/storage/service/ReliableForwarding.h

Ran tool

Read file: src/storage/service/ReliableForwarding.cc

Read file: src/storage/service/ReliableForwarding.cc


根据以上代码分析，我将为您提供一个详细的流程说明：

# 存储节点处理写IO与链式复制实现详解

## 1. 存储节点接收客户端写请求

当200KB的写IO请求到达存储节点时，首先由`StorageOperator::write`方法处理：

```cpp
CoTryTask<WriteRsp> StorageOperator::write(ServiceRequestContext &requestCtx,
                                           const WriteReq &req,
                                           net::IBSocket *ibSocket) {
  // 记录请求日志
  XLOGF(DBG1, "Received write request {} with tag {} to chunk {}", fmt::ptr(&req), req.tag, req.payload.key.chunkId);
  
  // 1. 根据chainId获取目标target
  auto targetResult = components_.targetMap.getByChainId(req.payload.key.vChainId);
  if (!targetResult) {
    rsp.result.lengthInfo = makeError(std::move(targetResult.error()));
    co_return rsp;
  }
  auto target = std::move(*targetResult);
  
  // 2. 转换为内部update请求，标记来源是客户端
  UpdateReq updateReq{req.payload, {}, req.tag, req.retryCount, req.userInfo, req.featureFlags};
  updateReq.options.fromClient = true;
  
  // 3. 执行可靠更新操作
  rsp.result = co_await components_.reliableUpdate.update(requestCtx, updateReq, ibSocket, target);
  co_return rsp;
}
```

## 2. 存储节点处理更新请求

当存储节点收到更新请求后，会通过`StorageOperator::handleUpdate`方法处理：

```cpp
CoTask<IOResult> StorageOperator::handleUpdate(ServiceRequestContext &requestCtx,
                                               UpdateReq &req,
                                               net::IBSocket *ibSocket,
                                               TargetPtr &target) {
  // 1. 验证目标节点
  if (req.options.fromClient && !target->isHead) {
    co_return makeError(StorageClientCode::kRoutingError, "non-head node receive a client update request");
  }
  
  // 2. 锁定chunk
  auto lockGuard = target->storageTarget->lockChunk(baton, req.payload.key.chunkId, fmt::to_string(req.tag));
  if (!lockGuard.locked()) {
    co_await lockGuard.lock();
  }
  
  // 3. 更新本地target
  auto buffer = components_.rdmabufPool.get();
  net::RDMARemoteBuf remoteBuf;
  ChunkEngineUpdateJob chunkEngineJob{};
  
  auto updateResult = co_await doUpdate(requestCtx,
                                       req.payload,
                                       req.options,
                                       req.featureFlags,
                                       target->storageTarget,
                                       ibSocket,
                                       buffer,
                                       remoteBuf,
                                       chunkEngineJob,
                                       !(req.options.fromClient && target->rejectCreateChunk));
                                       
  // 如果更新失败，返回错误
  if (!updateResult.lengthInfo) {
    co_return updateResult;
  }
  
  // 4. 转发给后继节点
  CommitIO commitIO;
  commitIO.key = req.payload.key;
  commitIO.commitVer = updateResult.updateVer;
  commitIO.isRemove = req.payload.isRemove();

  auto forwardResult = co_await components_.reliableForwarding
                          .forwardWithRetry(requestCtx, req, remoteBuf, chunkEngineJob, target, commitIO);
  
  // 5. 提交本地更改
  auto commitResult =
      co_await doCommit(requestCtx, commitIO, req.options, chunkEngineJob, req.featureFlags, target->storageTarget);
      
  commitResult.lengthInfo = updateResult.lengthInfo;
  commitResult.checksum = updateResult.checksum;
  
  co_return commitResult;
}
```

## 3. 存储节点申请chunk及更新数据块

### 3.1 Chunk创建和数据写入

当`doUpdate`被调用时，系统会创建或更新一个chunk：

```cpp
CoTask<IOResult> StorageOperator::doUpdate(ServiceRequestContext &requestCtx,
                                           const UpdateIO &updateIO,
                                           const UpdateOptions &updateOptions,
                                           uint32_t featureFlags,
                                           const std::shared_ptr<StorageTarget> &target,
                                           net::IBSocket *ibSocket,
                                           BufferPool::Buffer &buffer,
                                           net::RDMARemoteBuf &remoteBuf,
                                           ChunkEngineUpdateJob &chunkEngineJob,
                                           bool allowToAllocate) {
  // 创建更新任务
  UpdateJob job(requestCtx, updateIO, updateOptions, chunkEngineJob, target, allowToAllocate);
  
  // 处理数据传输
  if (updateIO.isWrite()) {
    // 分配缓冲区并通过RDMA读取客户端数据
    auto allocateResult = buffer.tryAllocate(updateIO.rdmabuf.size());
    job.state().data = allocateResult->ptr();
    remoteBuf = allocateResult->toRemoteBuf();
    
    // 通过RDMA从客户端读取数据
    auto readBatch = ibSocket->rdmaReadBatch();
    auto batchAddResult = readBatch.add(updateIO.rdmabuf, std::move(*allocateResult));
    auto postResult = co_await readBatch.post();
  }
  
  // 将UpdateJob提交给UpdateWorker执行
  co_await updateWorker_.enqueue(&job);
  co_await job.complete();
  
  co_return std::move(job.result());
}
```

### 3.2 UpdateWorker处理更新任务

UpdateWorker将任务分发给适当的线程执行：

```cpp
void UpdateWorker::run(Queue &queue) {
  while (true) {
    auto job = queue.dequeue();
    if (job == nullptr) break;
    
    // 调用StorageTarget的updateChunk方法执行实际更新
    job->target()->updateChunk(*job, bgExecutors_);
  }
}
```

### 3.3 StorageTarget执行chunk更新

StorageTarget根据配置选择使用ChunkEngine或ChunkReplica来更新数据：

```cpp
void StorageTarget::updateChunk(UpdateJob &job, folly::CPUThreadPoolExecutor &executor) {
  if (job.type() == UpdateType::COMMIT) {
    // 处理提交操作
    if (useChunkEngine()) {
      job.setResult(ChunkEngine::commit(*engine_, job, config_.kv_store().sync_when_write()));
    } else {
      job.setResult(ChunkReplica::commit(chunkStore_, job));
    }
  } else {
    // 处理更新操作
    auto result = useChunkEngine() 
        ? ChunkEngine::update(*engine_, job) 
        : ChunkReplica::update(chunkStore_, job, executor);
    job.setResult(std::move(result));
  }
}
```

### 3.4 ChunkReplica实现的更新操作

对于使用ChunkReplica的情况，更新过程如下：

```cpp
Result<uint32_t> ChunkReplica::update(ChunkStore &store, UpdateJob &job, folly::CPUThreadPoolExecutor &executor) {
  const auto &writeIO = job.updateIO();
  const auto &chunkId = writeIO.key.chunkId;
  const auto &state = job.state();
  auto &result = job.result();
  
  // 1. 获取元数据
  ChunkInfo chunkInfo;
  bool needCreateChunk = false;
  auto metaResult = store.get(chunkId);
  
  if (metaResult) {
    // 已存在的chunk
    chunkInfo = (*metaResult)->second;
  } else if (metaResult.error().code() == StorageCode::kChunkMetadataNotFound) {
    // 需要创建新chunk
    if (writeIO.isRemove()) {
      // 删除不存在的chunk，直接返回成功
      result.commitVer = result.updateVer = writeIO.updateVer;
      return 0;
    } else {
      // 创建新chunk
      needCreateChunk = true;
      chunkInfo.meta.chainVer = job.commitChainVer();
      chunkInfo.meta.chunkState = ChunkState::CLEAN;
      chunkInfo.meta.innerFileId.chunkSize = writeIO.chunkSize;
    }
  }
  
  // 2. 更新元数据
  ChunkMetadata &meta = chunkInfo.meta;
  meta.updateVer = writeIO.updateVer > 0 ? writeIO.updateVer : meta.updateVer + 1;
  meta.chunkState = ChunkState::DIRTY;
  meta.chainVer = job.commitChainVer();
  meta.timestamp = UtcClock::now();
  
  // 3. 存储元数据和创建chunk
  auto setResult = needCreateChunk 
      ? store.createChunk(chunkId, writeIO.chunkSize, chunkInfo, executor, job.allowToAllocate())
      : store.set(chunkId, chunkInfo, !skipPersist);
      
  // 4. 执行实际写入操作
  Result<uint32_t> writeResult = 0;
  if (writeIO.isWrite()) {
    // 执行写入操作
    writeResult = doRealWrite(chunkId, chunkInfo, state.data, writeIO.length, writeIO.offset);
  }
  
  // 5. 更新校验和
  auto checksumRes = updateChecksum(chunkInfo, writeIO, chunkSizeBeforeWrite, isAppendWrite);
  
  // 6. 完成写入
  meta.chunkState = ChunkState::CLEAN;
  setResult = store.set(chunkId, chunkInfo, !skipPersist);
  
  result.checksum = meta.checksum();
  result.commitVer = meta.commitVer;
  result.commitChainVer = meta.chainVer;
  
  return writeResult;
}
```

### 3.5 ChunkEngine实现的更新操作

对于使用ChunkEngine的情况，更新过程如下：

```cpp
Result<uint32_t> ChunkEngine::update(chunk_engine::Engine &engine, UpdateJob &job) {
  const auto &updateIO = job.updateIO();
  const auto &chunkId = updateIO.key.chunkId;
  const auto &state = job.state();
  auto &result = job.result();
  
  auto chainId = updateIO.key.vChainId.chainId;
  std::string key;
  key.reserve(sizeof(chainId) + chunkId.data().size());
  key.append((const char *)&chainId, sizeof(chainId));
  key.append(chunkId.data());
  
  // 准备更新请求
  chunk_engine::UpdateReq req{};
  req.is_truncate = updateIO.isTruncate();
  req.is_remove = updateIO.isRemove();
  req.is_syncing = updateOptions.isSyncing;
  req.update_ver = updateIO.updateVer;
  req.chain_ver = job.commitChainVer();
  req.length = updateIO.isWrite() ? updateIO.length : 0;
  req.offset = updateIO.isWrite() ? updateIO.offset : updateIO.length;
  req.data = reinterpret_cast<uint64_t>(state.data);
  
  // 执行更新操作
  std::string error{};
  auto chunk = engine.update_raw_chunk(toSlice(key), req, error);
  
  result.updateVer = result.commitVer = ChunkVer{req.out_commit_ver};
  result.commitChainVer = ChainVer{req.out_chain_ver};
  result.checksum = ChecksumInfo{ChecksumType::CRC32C, ~req.out_checksum};
  
  if (!error.empty()) {
    return makeError(req.out_error_code, std::move(error));
  }
  
  job.chunkEngineJob().set(engine, chunk);
  return updateIO.length;
}
```

## 4. 链式复制实现

链式复制由`ReliableForwarding`类实现，主要通过`forwardWithRetry`方法：

### 4.1 可靠转发过程

```cpp
CoTask<IOResult> ReliableForwarding::forwardWithRetry(ServiceRequestContext &requestCtx,
                                                      const UpdateReq &req,
                                                      const net::RDMARemoteBuf &rdmabuf,
                                                      const ChunkEngineUpdateJob &chunkEngineJob,
                                                      TargetPtr &target,
                                                      CommitIO &commitIO,
                                                      bool allowOutdatedChainVer) {
  // 使用指数退避策略进行重试
  ExponentialBackoffRetry retry(config_.retry_first_wait().asMs(),
                               config_.retry_max_wait().asMs(),
                               config_.retry_total_time().asMs());
                               
  for (uint32_t retryCount = 0; !stopped_; ++retryCount) {
    auto waitTime = retry.getWaitTime();
    
    // 获取最新的target信息
    auto targetResult = components_.targetMap.getByChainId(req.payload.key.vChainId, allowOutdatedChainVer);
    target = std::move(*targetResult);
    
    // 执行转发
    auto ioResult = co_await forward(req, retryCount, rdmabuf, chunkEngineJob, target, commitIO, waitTime);
    
    if (bool(ioResult.lengthInfo) || ioResult.lengthInfo.error().code() == StorageCode::kNoSuccessorTarget) {
      co_return ioResult;
    }
    
    // 处理错误情况，决定是否重试
    auto code = ioResult.lengthInfo.error().code();
    if (!allowOutdatedChainVer && code == StorageClientCode::kRoutingVersionMismatch) {
      co_return ioResult;
    }
    
    // 如果超时或需要重试，等待一段时间
    if (waitTime.count() == 0) {
      co_return ioResult;
    } else if (code != RPCCode::kTimeout) {
      // 等待一段时间后重试
      for (auto elapsed = 0ms; elapsed < waitTime && !stopped_; elapsed += checkInterval) {
        // 尝试更新target信息
        auto targetResult = components_.targetMap.getByChainId(req.payload.key.vChainId, allowOutdatedChainVer);
        target = std::move(*targetResult);
        if (!target->successor.has_value()) {
          break;
        }
        co_await folly::coro::sleep(std::min(checkInterval, waitTime - elapsed));
      }
    }
  }
  
  co_return makeError(RPCCode::kRequestRefused, "req is refused because of stopping");
}
```

### 4.2 转发给后继节点

`forward`方法负责将请求转发给链中的下一个节点：

```cpp
CoTask<IOResult> ReliableForwarding::forward(const UpdateReq &req,
                                            uint32_t retryCount,
                                            const net::RDMARemoteBuf &rdmabuf,
                                            const ChunkEngineUpdateJob &chunkEngineJob,
                                            TargetPtr &target,
                                            CommitIO &commitIO,
                                            std::chrono::milliseconds timeout) {
  // 如果没有后继节点，直接返回
  if (!target->successor.has_value()) {
    commitIO.commitChainVer = target->vChainId.chainVer;
    co_return makeError(StorageCode::kNoSuccessorTarget);
  }
  
  // 执行实际转发
  auto ioResult = co_await doForward(req, rdmabuf, chunkEngineJob, retryCount, *target, commitIO.isSyncing, timeout);
  
  if (ioResult.lengthInfo) {
    // 更新提交版本
    commitIO.commitVer = ioResult.commitVer;
    commitIO.commitChainVer = ioResult.commitChainVer;
    
    // 检查链版本
    if (ioResult.commitChainVer > target->vChainId.chainVer) {
      co_return makeError(StorageCode::kChainVersionMismatch, "the remote obtains a higher chain version");
    }
  }
  
  co_return ioResult;
}
```

### 4.3 实际执行转发

`doForward`方法执行实际的转发操作：

```cpp
CoTask<IOResult> ReliableForwarding::doForward(const UpdateReq &req,
                                              const net::RDMARemoteBuf &rdmabuf,
                                              const ChunkEngineUpdateJob &chunkEngineJob,
                                              uint32_t retryCount,
                                              const Target &target,
                                              bool &isSyncing,
                                              std::chrono::milliseconds timeout) {
  // 准备转发请求
  UpdateReq updateReq = req;
  updateReq.options.fromClient = false;  // 标记请求不是来自客户端
  updateReq.retryCount = retryCount;
  updateReq.payload.rdmabuf = rdmabuf;
  updateReq.payload.key.vChainId.chainVer = target.vChainId.chainVer;
  
  // 检查后继节点是否在同步状态
  isSyncing = target.successor->targetInfo.publicState == hf3fs::flat::PublicTargetState::SYNCING;
  if (isSyncing) {
    updateReq.options.isSyncing = true;
    updateReq.options.commitChainVer = target.vChainId.chainVer;
  }
  
  // 如果是同步写入且需要读取数据，则读取整个chunk
  bool readForSyncing = req.payload.isWriteTruncateExtend() && isSyncing &&
                       (req.options.isSyncing || req.payload.length != req.payload.chunkSize);
  if (readForSyncing) {
    // 读取整个chunk以便同步
    // ... (读取逻辑)
  }
  
  // 获取后继节点地址
  auto addrResult = target.getSuccessorAddr();
  
  // 设置请求选项
  net::UserRequestOptions reqOptions;
  reqOptions.timeout = Duration{timeout};
  
  // 发送更新请求到后继节点
  auto updateResult = co_await components_.messenger.update(*addrResult, updateReq, &reqOptions);
  
  if (updateResult && bool(updateResult->result.lengthInfo)) {
    // 检查链版本
    if (target.vChainId.chainVer < updateResult->result.commitChainVer) {
      co_return makeError(StorageCode::kChainVersionMismatch, "chain version local < remote");
    }
    
    // 记录统计信息
    auto length = *updateResult->result.lengthInfo;
    if (isSyncing) {
      forwardSyncingBytes.addSample(length);
    } else {
      forwardWriteBytes.addSample(length);
    }
  } else {
    // 处理错误
    XLOGF(ERR, "forward failed, req {}, result {}", updateReq, updateResult->result);
  }
  
  co_return updateResult->result;
}
```

## 5. 完整流程实例（200KB写入，64KB块大小）

以下是整个流程的详细示例，假设chunk大小为64KB，写入200KB数据：

### 5.1 客户端发送写请求

1. 客户端发送200KB大小的写IO请求到头节点（Head Target）
2. 请求包含数据的RDMA缓冲区地址，以及目标chunk信息

### 5.2 头节点处理请求

1. 头节点通过`StorageOperator::write`接收客户端请求
2. 验证目标有效性，确保是链的头节点
3. 尝试锁定目标chunk

### 5.3 数据接收和存储

1. 头节点通过RDMA从客户端读取200KB数据到本地缓冲区
2. 创建`UpdateJob`任务并提交给`UpdateWorker`
3. `UpdateWorker`调用`StorageTarget::updateChunk`处理更新

### 5.4 Chunk分配和创建

1. 如果chunk不存在，创建新的chunk：
   - 针对200KB数据，需要创建多个chunk（4个）
   - 每个chunk大小为64KB（第四个只使用8KB）
   - 设置元数据：updateVer、chainVer等

2. 对于ChunkReplica实现：
   ```cpp
   // 创建新chunk
   auto setResult = store.createChunk(chunkId, writeIO.chunkSize, chunkInfo, executor, job.allowToAllocate());
   ```

3. 对于ChunkEngine实现：
   ```cpp
   // 通过Engine创建chunk
   auto chunk = engine.update_raw_chunk(toSlice(key), req, error);
   ```

### 5.5 数据写入到本地存储

1. 对于ChunkReplica实现：
   ```cpp
   // 执行实际写入
   writeResult = doRealWrite(chunkId, chunkInfo, state.data, writeIO.length, writeIO.offset);
   ```

2. 对于ChunkEngine实现，数据直接通过`req.data`指向的缓冲区写入

3. 更新校验和：
   - 如果是新chunk，直接使用写入数据的校验和
   - 如果是追加写入，组合现有校验和和新数据校验和
   - 如果是随机写入，可能需要读取现有数据计算新校验和

### 5.6 更新元数据

1. 更新chunk元数据：
   ```cpp
   meta.updateVer = writeIO.updateVer > 0 ? writeIO.updateVer : meta.updateVer + 1;
   meta.chunkState = ChunkState::DIRTY;  // 标记为脏状态
   meta.chainVer = job.commitChainVer();
   meta.timestamp = UtcClock::now();
   ```

2. 持久化元数据：
   ```cpp
   auto setResult = store.set(chunkId, chunkInfo, !skipPersist);
   ```

### 5.7 链式复制过程

1. 头节点通过`ReliableForwarding::forwardWithRetry`将更新请求转发到第二个节点：
   ```cpp
   auto forwardResult = co_await components_.reliableForwarding
                         .forwardWithRetry(requestCtx, req, remoteBuf, chunkEngineJob, target, commitIO);
   ```

2. 构建转发请求：
   ```cpp
   UpdateReq updateReq = req;
   updateReq.options.fromClient = false;  // 标记不是客户端请求
   updateReq.payload.rdmabuf = rdmabuf;   // 设置RDMA缓冲区
   ```

3. 发送更新请求到下一个节点：
   ```cpp
   auto updateResult = co_await components_.messenger.update(*addrResult, updateReq, &reqOptions);
   ```

4. 下一个节点执行相同的更新过程：
   - 从前一个节点读取数据
   - 更新本地存储
   - 继续转发给下一个节点（如果有）

5. 链中的最后一个节点没有后继节点，返回`kNoSuccessorTarget`错误

### 5.8 提交阶段

1. 所有节点成功更新后，开始从最后一个节点向前提交：
   ```cpp
   auto commitResult =
       co_await doCommit(requestCtx, commitIO, req.options, chunkEngineJob, req.featureFlags, target->storageTarget);
   ```

2. 提交操作将chunk状态从DIRTY改为CLEAN：
   ```cpp
   meta.chunkState = ChunkState::CLEAN;
   meta.commitVer = commitIO.commitVer;
   ```

3. 对于ChunkReplica：
   ```cpp
   auto metaResult = store.set(chunkId, chunkInfo);
   ```

4. 对于ChunkEngine：
   ```cpp
   engine.commit_raw_chunk(chunk, sync, error);
   ```

5. 头节点收到所有确认后，向客户端返回写入成功响应

## 6. 总结

1. **写入流程**：
   - 客户端发送写请求到头节点
   - 头节点通过RDMA读取客户端数据
   - 头节点创建或更新chunk，并写入数据
   - 头节点将更新转发给链中的下一个节点
   - 链式复制直到所有节点完成更新
   - 从末尾节点开始向前提交更改
   - 头节点返回成功响应给客户端

2. **Chunk申请**：
   - 当chunk不存在时，系统会根据请求的chunk大小创建新chunk
   - 对于200KB的写入，系统会创建4个64KB的chunk
   - chunk创建包括分配存储空间和初始化元数据

3. **元数据更新**：
   - 每个chunk包含重要元数据：updateVer, commitVer, chainVer, chunkState等
   - 写入过程中，chunk状态标记为DIRTY
   - 提交成功后，状态更新为CLEAN或COMMIT
   - 元数据更新是原子性的，确保一致性

4. **链式复制**：
   - 数据从链头向链尾传递
   - 确认从链尾向链头传递
   - 每个节点独立验证数据完整性(校验和)
   - 支持重试机制，确保高可用性
   - 支持节点同步状态，处理节点恢复场景

这个设计确保了数据写入的一致性、可靠性和容错性，同时通过链式复制提供了良好的扩展性和性能。
