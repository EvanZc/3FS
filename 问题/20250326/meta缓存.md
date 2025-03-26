# Meta的缓存

---

## AclCache在源数据节点是如何被初始化的，它每个实例会占用多少内存？

---

让我帮你分析AclCache的初始化和内存占用情况。

首先，让我们查看AclCache的初始化相关代码：


Ran tool
根据代码分析，AclCache的初始化和内存占用情况如下：

### 1. 初始化过程

AclCache在MetaStore构造函数中被初始化：

```cpp
// src/meta/store/MetaStore.h
MetaStore(const Config &config,
          analytics::StructuredTraceLog<MetaEventTrace> &metaEventTraceLog,
          std::shared_ptr<Distributor> distributor,
          std::shared_ptr<InodeIdAllocator> inodeAlloc,
          std::shared_ptr<ChainAllocator> chainAlloc,
          std::shared_ptr<FileHelper> fileHelper,
          std::shared_ptr<SessionManager> sessionManager,
          std::shared_ptr<GcManager> gcManager)
    : config_(config),
      metaEventTraceLog_(metaEventTraceLog),
      distributor_(distributor),
      inodeAlloc_(inodeAlloc),
      chainAlloc_(chainAlloc),
      fileHelper_(fileHelper),
      sessionManager_(sessionManager),
      gcManager_(gcManager),
      aclCache_(2 << 20 /* 2m acl */) {}
```

### 2. 内存占用分析

AclCache的内存占用由以下几个部分组成：

1. 分片数量：
```cpp
// src/meta/components/AclCache.h
static constexpr auto kNumShards = 32u;
```

2. 每个分片的大小：
```cpp
// src/meta/components/AclCache.h
shardedMaps_.emplace_back(
    folly::EvictingCacheMap<InodeId, CacheEntry>(std::max(cacheSize / kNumShards, 1ul << 10), 128));
```

3. 每个缓存项的大小：
```cpp
// src/meta/components/AclCache.h
struct CacheEntry {
    SteadyTime timestamp;  // 8 bytes
    Acl acl;              // 包含权限信息
};
```

### 3. 内存计算

1. 总缓存大小：
- 默认大小为 2^21 (2 << 20) = 2,097,152 个条目
- 每个分片大小 = 2,097,152 / 32 = 65,536 个条目

2. 每个缓存项大小：
- SteadyTime: 8 bytes
- Acl: 包含权限信息，假设为 32 bytes
- InodeId: 8 bytes
- 总计：约 48 bytes 每项

3. 总内存占用：
- 每个分片：65,536 * 48 = 3,145,728 bytes ≈ 3MB
- 32个分片总内存：3MB * 32 = 96MB

### 4. 缓存特性

1. 分片设计：
```cpp
CacheMap &getShard(InodeId inode) {
    auto shardId = inode.u64() % kNumShards;
    return shardedMaps_[shardId];
}
```

2. 过期机制：
```cpp
auto deadline = cached->timestamp + ttl * folly::Random::randDouble(0.8, 1.0);
if (deadline < SteadyClock::now()) {
    miss.addSample(1);
    return std::nullopt;
}
```

3. 缓存更新：
```cpp
void set(InodeId inode, Acl acl) {
    auto &shard = getShard(inode);
    shard.lock()->set(inode, {SteadyClock::now(), acl});
}
```

### 5. 性能优化

1. 分片减少锁竞争：
- 使用32个分片分散访问压力
- 每个分片有独立的锁

2. LRU淘汰机制：
- 使用EvictingCacheMap自动淘汰最久未使用的项
- 每个分片最多128个条目

3. 随机过期时间：
- 使用随机因子(0.8-1.0)避免缓存雪崩

总结：AclCache在Meta节点中是一个重要的缓存组件，用于加速权限检查。每个实例默认占用约96MB内存，通过分片和LRU机制实现了高效的缓存管理。缓存大小可以通过配置调整，默认支持200万个ACL缓存项。

---

## 什么时候会访问acl缓存，请用代码举例

---

让我帮你分析AclCache的访问场景。主要是在路径解析和权限检查时会访问AclCache。

让我们看几个具体的代码示例：

1. 路径解析时的ACL缓存访问：
```cpp
// src/meta/store/PathResolve.cc
static CoTryTask<std::variant<std::pair<InodeId, Acl>, Inode, DirEntry>> loadParentAcl(
    IReadOnlyTransaction &txn,
    AclCache &cache,
    const UserInfo &user,
    const std::variant<InodeId, DirEntry> &parent,
    Duration cacheTime) {
  
  if (parentId == InodeId::root()) {
    parentInfo = DirEntry::root();
  } else {
    // 首先尝试从缓存获取ACL
    auto cached = cache.get(parentId, cacheTime);
    if (cached.has_value()) {
      parentInfo = std::pair<InodeId, Acl>{parentId, *cached};
    } else {
      // 缓存未命中，从数据库加载
      auto loadResult = (co_await Inode::snapshotLoad(txn, parentId)).then(checkMetaFound<Inode>);
      CO_RETURN_ON_ERROR(loadResult);
      if (cacheTime.count() != 0 && loadResult->isDirectory()) {
        // 将ACL存入缓存
        cache.set(parentId, loadResult->acl);
      }
      parentInfo = std::move(*loadResult);
    }
  }
}
```

2. 创建文件时的ACL缓存访问：
```cpp
// src/meta/store/ops/BatchOperation.cc
auto acl = Acl(req_.user.uid,
               req_.user.gid,
               Permission(req_.perm & ALLPERMS),
               IFlags(parent->acl.iflags & FS_FL_INHERITABLE));
if (parent->acl.perm & S_ISGID) {
  // 继承父目录的组ID
  acl.gid = parent->acl.gid;
  acl.perm = Permission(acl.perm | S_ISGID);
}
```

3. 修改文件属性时的ACL缓存失效：
```cpp
// src/meta/store/ops/SetAttr.cc
void finish(const Result<SetAttrRsp> &result) override {
  Operation<SetAttrRsp>::finish(result);
  if (!result.hasError()) {
    // 当修改了uid、gid、权限或标志位时，使ACL缓存失效
    if (req_.uid || req_.gid || req_.perm || req_.iflags) 
      aclCache().invalid(result->stat.id);
  }
}
```

4. 目录操作时的ACL缓存访问：
```cpp
// src/meta/store/ops/Mkdirs.cc
auto parent = co_await resolveResult->getParentInode(txn);
CO_RETURN_ON_ERROR(parent);
// 检查父目录的写入权限
CO_RETURN_ON_ERROR(parent->acl.checkPermission(req_.user, AccessType::WRITE));
// 检查目录锁
CO_RETURN_ON_ERROR(parent->asDirectory().checkLock(req_.client));
```

5. 递归删除时的ACL缓存访问：
```cpp
// src/meta/store/DirEntry.h
static CoTryTask<Void> recursiveCheckRmPerm(IReadWriteTransaction &txn,
                                          InodeId parent,
                                          flat::UserInfo user,
                                          int32_t limit,
                                          size_t listBatchSize) {
  // ... 
  for (auto &entry : list->entries) {
    if (!entry.isDirectory()) {
      continue;
    }
    auto &acl = *entry.dirAcl;
    // 检查递归删除权限
    if (auto res = acl.checkRecursiveRmPerm(user, false); res.hasError()) {
      auto msg = fmt::format("user {} recursive remove {}, found {} without permission, msg {}",
                             user.uid,
                             parent,
                             entry,
                             res.error().message());
      XLOG(ERR, msg);
      co_return makeError(MetaCode::kNoPermission, msg);
    }
  }
}
```

6. 文件打开时的ACL缓存访问：
```cpp
// src/meta/store/ops/Open.cc
// 检查权限
CO_RETURN_ON_ERROR(inode.acl.checkPermission(req_.user, req_.flags.accessType()));

// 检查文件空洞
if (rdonly && inode.asFile().hasHole() && checkHole) {
    XLOGF(WARN, "Inode {} contains hole, don't allow O_RDONLY", inode.id);
    co_return makeError(MetaCode::kFileHasHole);
}
```

主要访问场景总结：

1. **路径解析**：
   - 解析文件路径时需要检查每个目录的访问权限
   - 使用ACL缓存减少数据库访问

2. **文件操作**：
   - 创建文件/目录时继承父目录的ACL
   - 修改文件属性时使ACL缓存失效
   - 打开文件时检查访问权限

3. **目录操作**：
   - 创建目录时继承父目录的ACL
   - 递归删除时检查权限
   - 目录锁定时的权限检查

4. **权限变更**：
   - 修改文件/目录权限时使ACL缓存失效
   - 修改所有者/组时使ACL缓存失效

这些场景都利用了ACL缓存来提高性能，避免频繁访问数据库。缓存的设计考虑了并发访问（通过分片）和一致性（通过缓存失效机制）。

---

## CoTryTask<ResolveResult> PathResolveOp::path(InodeId parent, const Path &path) 这个函数在什么情况下会被调用？

---

让我帮你分析`PathResolveOp::path`函数的调用场景。这个函数主要用于解析文件路径，让我们看看它的调用链：

1. 首先看这个函数的主要入口：
```cpp
// src/meta/store/PathResolve.h
class PathResolveOp {
public:
    // 主要入口点
    CoTryTask<ResolveResult> path(const PathAt &path, AtFlags flags) {
        if (!path.path.has_value()) {
            co_return makeError(StatusCode::kInvalidArg, "path not set");
        } else {
            co_return co_await this->path(path.parent, *path.path, flags.followLastSymlink());
        }
    }
};
```

2. 这个函数在以下操作中被调用：

```cpp
// src/meta/store/ops/Open.cc
CoTryTask<OpenRsp> OpenOp::run(IReadWriteTransaction &txn) {
    // 打开文件时解析路径
    auto resolveResult = co_await resolve(txn, req_.user).path(req_.path);
}

// src/meta/store/ops/Create.cc
CoTryTask<CreateRsp> CreateOp::run(IReadWriteTransaction &txn) {
    // 创建文件时解析路径
    auto resolveResult = co_await resolve(txn, req_.user).path(req_.path);
}

// src/meta/store/ops/Mkdirs.cc
CoTryTask<MkdirsRsp> MkdirsOp::run(IReadWriteTransaction &txn) {
    // 创建目录时解析路径
    auto resolveResult = co_await resolve(txn, req_.user).path(req_.path);
}

// src/meta/store/ops/Remove.cc
CoTryTask<RemoveRsp> RemoveOp::run(IReadWriteTransaction &txn) {
    // 删除文件/目录时解析路径
    auto resolveResult = co_await resolve(txn, req_.user).path(req_.path);
}

// src/meta/store/ops/Rename.cc
CoTryTask<RenameRsp> RenameOp::run(IReadWriteTransaction &txn) {
    // 重命名时解析源路径和目标路径
    auto srcResolveResult = co_await resolve(txn, req_.user).path(req_.src);
    auto dstResolveResult = co_await resolve(txn, req_.user).path(req_.dst);
}

// src/meta/store/ops/List.cc
CoTryTask<ListRsp> ListOp::run(IReadOnlyTransaction &txn) {
    // 列出目录内容时解析路径
    auto resolveResult = co_await resolve(txn, req_.user).path(req_.path);
}

// src/meta/store/ops/Stat.cc
CoTryTask<StatRsp> StatOp::run(IReadOnlyTransaction &txn) {
    // 获取文件状态时解析路径
    auto resolveResult = co_await resolve(txn, req_.user).path(req_.path);
}
```

3. 函数的主要功能：

```cpp
// src/meta/store/PathResolve.cc
CoTryTask<ResolveResult> PathResolveOp::path(InodeId parent, const Path &path) {
    XLOGF(DBG, "Resolve path {}/{}", parent, path);
    auto begin = path.begin();
    // 解析路径范围
    auto resolveResult = co_await this->pathRange(parent, begin, path.end());
    CO_RETURN_ON_ERROR(resolveResult);

    // 记录解析结果
    XLOGF(DBG,
          "Resolve path {}, {} components, {} found, {} missing",
          path,
          std::distance(path.begin(), path.end()),
          std::distance(path.begin(), begin),
          std::distance(begin, path.end()));

    // 检查路径是否完整解析
    if (begin == path.end() || ++begin == path.end()) {
        co_return resolveResult;
    } else {
        // 路径中间部分缺失，返回未找到错误
        co_return makeError(MetaCode::kNotFound);
    }
}
```

4. 调用场景总结：

- **文件操作**：
  - 打开文件 (open)
  - 创建文件 (create)
  - 删除文件 (remove)
  - 重命名文件 (rename)
  - 获取文件状态 (stat)

- **目录操作**：
  - 创建目录 (mkdir)
  - 删除目录 (remove)
  - 列出目录内容 (list)
  - 重命名目录 (rename)

- **路径解析**：
  - 解析完整路径
  - 处理相对路径
  - 处理符号链接
  - 验证路径存在性

5. 重要特性：

```cpp
// src/meta/store/PathResolve.h
class PathResolveOp {
private:
    IReadOnlyTransaction &txn_;
    const UserInfo &user_;
    AclCache &aclCache_;
    Path *trace_;
    size_t depth_;
    size_t symlinkCnt_;
    size_t maxSymlinkCount_;
    size_t maxSymlinkDepth_;
    Duration aclCacheTime_;
    size_t pathComponents_;
};
```

- 支持符号链接解析
- 使用ACL缓存优化权限检查
- 支持路径跟踪
- 限制符号链接深度
- 记录路径组件数量

这个函数是文件系统路径解析的核心组件，几乎所有涉及文件路径的操作都会调用它。它负责将用户提供的路径转换为文件系统中的实际位置，并确保访问权限的正确性。

---

## 读IO会访问ACL缓存吗？

---

让我帮你分析读IO时是否会访问ACL缓存。

首先，让我们看看读IO相关的代码：


Ran tool
根据代码分析，读IO操作确实会访问ACL缓存。让我详细解释读IO时ACL缓存的访问流程：

1. **读IO操作入口**：
   - 当执行读IO操作时，首先会调用 `OpenOp` 类来处理文件打开请求
   - 在 `OpenOp::openExistsFile` 方法中，会进行权限检查：
   ```cpp
   // check permission
   if (req_.flags.accessType() != AccessType::READ && (inode.acl.iflags & FS_IMMUTABLE_FL)) {
     co_return makeError(MetaCode::kNoPermission, fmt::format("FS_IMMUTABLE_FL set on inode {}", inode.id));
   }
   CO_RETURN_ON_ERROR(inode.acl.checkPermission(req_.user, req_.flags.accessType()));
   ```

2. **路径解析时的ACL缓存访问**：
   - 在 `PathResolveOp` 中，当解析文件路径时会访问ACL缓存
   - 通过 `loadParentAcl` 函数获取父目录的ACL信息：
   ```cpp
   auto cached = cache.get(parentId, cacheTime);
   if (cached.has_value()) {
     parentInfo = std::pair<InodeId, Acl>{parentId, *cached};
   } else {
     // 如果缓存未命中，则从数据库加载
     auto loadResult = (co_await Inode::snapshotLoad(txn, parentId)).then(checkMetaFound<Inode>);
     if (cacheTime.count() != 0 && loadResult->isDirectory()) {
       cache.set(parentId, loadResult->acl);
     }
     parentInfo = std::move(*loadResult);
   }
   ```

3. **ACL缓存的特点**：
   - ACL缓存使用分片设计（32个分片）来减少锁竞争
   - 每个缓存项包含时间戳和ACL信息
   - 缓存有TTL（Time To Live）机制，默认缓存时间为5秒
   - 使用 `EvictingCacheMap` 实现LRU（最近最少使用）淘汰策略

4. **缓存命中率监控**：
   - 系统会记录ACL缓存的命中率：
   ```cpp
   static monitor::CountRecorder hit("meta_server.aclcache_hit");
   static monitor::CountRecorder miss("meta_server.aclcache_miss");
   ```

5. **缓存失效场景**：
   - 当文件属性（如权限、所有者等）发生变化时，ACL缓存会被失效
   - 在 `SetAttrOp` 的 `finish` 方法中：
   ```cpp
   if (req_.uid || req_.gid || req_.perm || req_.iflags) 
     aclCache().invalid(result->stat.id);
   ```

总结：
1. 读IO操作会访问ACL缓存，主要用于权限检查
2. ACL缓存的设计目标是提高权限检查的性能，减少数据库访问
3. 缓存采用分片设计，支持并发访问
4. 系统会监控缓存的命中率，以便评估缓存效果
5. 当文件属性发生变化时，相关ACL缓存会被自动失效，保证数据一致性
