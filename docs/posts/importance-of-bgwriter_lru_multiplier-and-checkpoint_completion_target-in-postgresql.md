---
date: 2024-12-13
categories:
  - Bgwriter
  - Checkpointer
  - Settings
---

# PostgreSQL 中 `bgwriter_lru_multiplier` 和 `checkpoint_completion_target` 的重要性

这些参数在管理 bgwriter 和 checkpoint 操作起着至关重要的作用。以下是它们的工作原理并可以通过示例进行计算：
<!-- more -->

## `bgwriter_lru_multiplier`

- **目标：**
    - 控制 bgwriter 进程尝试维护可重用缓冲区的积极程度。
    - 如果可用空间不足，bgwriter 进程将尝试刷写更多的缓冲区，并且该参数决定它将清理多少个额外的缓冲区。
  
- **工作原理：**
	- bgwriter 进程根据当前缓冲区使用情况计算每轮应该刷写多少个缓冲区。
	- 该计算将考虑需要刷写的缓冲区数据并乘以 `bgwriter_lru_multiplier` 以决定总共需要刷写的缓冲区数量。
	
- **计算示例：**
	- 假设您有如下配置：
		- `bgwriter_lru_maxpages = 1000`
		- `bgwriter_lru_multiplier = 2.0`
		- 系统估计需要清理 500 个缓冲区来维护空闲缓冲池。

bgwriter 进程将尝试刷写的缓冲区：

```
buffers_to_write = estimated_needed_buffers * bgwriter_lru_multiplier
  = 500 * 2.0
  = 1000 buffers

# 如果 `bgwriter_lru_maxpages = 1000`，那么在每一轮中，bgwriter 进程将尝试最多
# 刷写 1000 个缓冲区，与允许的最大页面数保持一致。
```

## `checkpoint_completion_target`

- **目标：**
	- 确定 checkpoint 应分布在 `checkpoint_timeout` 的比例。
	- 通过写入操作分散到较长的时间段内（而非一次性写入）来缓解 I/O 压力。
	
- **工作原理：**
	- 值越高意味着 checkpoint 的 I/O 操作在 `checkpoint_timeout` 周期内分布越均匀。
	- checkpoint 写入的实际持续时间为 `checkpoint_timeout * checkpoint_completion_target`。

- **计算示例：**
	- 假设您有如下配置：
		- `checkpoint_timeout = 15 minutes`
		- `checkpoint_completion_target = 0.9`

Checkpoint 目标完成时间为：

```
target_duration = checkpoint_timeout * checkpoint_completion_target
  = 15 minutes * 0.9
  = 13.5 minutes
  
# 这意味着 PostgreSQL 将尝试将 checkpoint 写入分散到 13.5 分钟内，以减少 I/O 影响。
```

## 示例配置

**最差配置：**

```
# Background Writer Parameters
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 10
bgwriter_lru_multiplier = 1.0

# Checkpointer Parameters
checkpoint_timeout = 5min
checkpoint_completion_target = 0.5
max_wal_size = 1GB
min_wal_size = 80MB

# 问题： bgwriter 每轮写入的页面太少（10 页），导致 backend 写入活动频繁。
# checkpoint 过于频繁（每 5 分钟一次），完成速度快（2.5 分钟内），导致 I/O 峰值。
```

**改进的配置：**

```
# Background Writer Parameters
bgwriter_delay = 100ms
bgwriter_lru_maxpages = 1000
bgwriter_lru_multiplier = 2.0

# Checkpointer Parameters
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9
max_wal_size = 4GB
min_wal_size = 1GB

# 收益：bgwriter 每轮写入最大可到 1000 页，并尝试维护更大的可重用缓冲区（计算结果为 2000）。
# checkpoint 的频率更低，时间更长（13.5 分钟），从而降低了 I/O 峰值。
```

## 总结

合理的调整 `bgwriter_lru_multiplier` 和 `checkpoint_completion_target` 对于平衡 PostgreSQL 的性能和稳定性至关重要。

`bgwriter_lru_multiplier` 有助于确定 bgwriter 清理缓冲区的积极程度，而 `checkpoint_completion_target` 可确保 checkpoint 分散以最大限度地减少 I/O 峰值。

根据您的工作负载和系统功能优化这些参数，您可以实现更高效、响应更快的 PostgreSQL 环境。

> 作者：Sheikh Wasiu Al Hasib<br>
> 原文：[https://medium.com/@wasiualhasib/importance-of-bgwriter-lru-multiplier-and-checkpoint-completion-target-in-postgresql-79cf76fc04d3](https://medium.com/@wasiualhasib/importance-of-bgwriter-lru-multiplier-and-checkpoint-completion-target-in-postgresql-79cf76fc04d3)

