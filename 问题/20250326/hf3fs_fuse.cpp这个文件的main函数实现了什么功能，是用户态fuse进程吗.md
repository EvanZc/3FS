# fuse

--- 

让我帮你分析`hf3fs_fuse.cpp`中的main函数功能。

### 主要功能

1. 初始化FUSE文件系统：
```cpp
int main(int argc, char *argv[]) {
    // 解析命令行参数
    // 设置FUSE选项
    // 初始化存储客户端
    // 启动FUSE文件系统
}
```

2. 命令行参数处理：
```cpp
// 支持的参数包括：
- mount_point: 挂载点路径
- storage_url: 存储URL
- storage_token: 存储令牌
- storage_region: 存储区域
- storage_endpoint: 存储端点
- storage_use_https: 是否使用HTTPS
- storage_verify_ssl: 是否验证SSL
- storage_max_connections: 最大连接数
- storage_connect_timeout: 连接超时时间
- storage_request_timeout: 请求超时时间
- storage_max_retries: 最大重试次数
- storage_retry_delay: 重试延迟时间
- storage_retry_max_delay: 最大重试延迟时间
- storage_retry_backoff: 重试退避时间
- storage_retry_max_backoff: 最大重试退避时间
- storage_retry_jitter: 重试抖动时间
- storage_retry_max_jitter: 最大重试抖动时间
- storage_retry_max_attempts: 最大重试尝试次数
- storage_retry_max_duration: 最大重试持续时间
- storage_retry_max_interval: 最大重试间隔时间
- storage_retry_max_interval_jitter: 最大重试间隔抖动时间
- storage_retry_max_interval_backoff: 最大重试间隔退避时间
- storage_retry_max_interval_backoff_jitter: 最大重试间隔退避抖动时间
- storage_retry_max_interval_backoff_max: 最大重试间隔退避最大值
- storage_retry_max_interval_backoff_min: 最大重试间隔退避最小值
- storage_retry_max_interval_backoff_factor: 最大重试间隔退避因子
- storage_retry_max_interval_backoff_jitter_factor: 最大重试间隔退避抖动因子
- storage_retry_max_interval_backoff_max_factor: 最大重试间隔退避最大值因子
- storage_retry_max_interval_backoff_min_factor: 最大重试间隔退避最小值因子
- storage_retry_max_interval_backoff_factor_factor: 最大重试间隔退避因子因子
- storage_retry_max_interval_backoff_jitter_factor_factor: 最大重试间隔退避抖动因子因子
- storage_retry_max_interval_backoff_max_factor_factor: 最大重试间隔退避最大值因子因子
- storage_retry_max_interval_backoff_min_factor_factor: 最大重试间隔退避最小值因子因子
- storage_retry_max_interval_backoff_factor_factor_factor: 最大重试间隔退避因子因子因子
- storage_retry_max_interval_backoff_jitter_factor_factor_factor: 最大重explicitly specified.
