# 文档修正记录 (Correction Log)

**修正日期**: 2026-05-15
**修正原因**: 基于代码逐行验证发现的问题

---

## 修正 1: 09-complete-write-path-deep-analysis.md

### 1.1 WriteRows 行号修正
- **位置**: 文档第 30 行、62 行
- **原内容**: `shard.go:468`
- **修正后**: `shard.go:512`
- **偏差**: 44 行
- **原因**: 代码重构导致函数位置变更

### 1.2 MemTable 结构体行号修正
- **位置**: 文档第 12 行、307 行
- **原内容**: `table.go:57`
- **修正后**: `table.go:297`
- **偏差**: 240 行
- **原因**: 文件结构重组，MemTable 定义移动

### 1.3 WAL writeBinary 函数签名修正
- **位置**: 文档第 225 行、246 行
- **原内容**: 
  ```go
  func (w *wal) writeBinary(data []byte, typ WriteWalType, maxTime int64) error
  ```
- **修正后**:
  ```go
  func (l *WAL) Write(rows []byte, typ WalRecordType, maxRowTime int64) error
  func (l *WAL) writeBinary(walRecord *walRecord) error
  ```
- **变更说明**:
  - 公开方法从 `writeBinary` 改为 `Write`
  - 参数从直接传递 `[]byte` 改为使用 `*walRecord` 结构体
  - 内部 `writeBinary` 增加了 Snappy 压缩逻辑
  - WAL 记录格式变更：增加压缩层

---

## 修正 2: README-analysis-summary.md

### 2.1 WriteRows 引用修正
- **位置**: 文档第 61 行
- **原内容**: `WriteRows (shard.go:468)`
- **修正后**: `WriteRows (shard.go:512)`

### 2.2 WAL Write 引用修正
- **位置**: 文档第 65 行
- **原内容**: `writeToWal (WAL 追加：engine/wal.go:218)`
- **修正后**: `writeToWal (WAL 追加：engine/wal.go:248)`

### 2.3 删除重复代码块
- **位置**: 文档第 79-96 行
- **操作**: 删除重复的 WriteRows 调用链描述
- **原因**: 同一文档中存在两段相同的调用链描述

---

## 验证结果

### 修正后验证状态
| 文档 | 修正项数 | 验证状态 |
|------|---------|---------|
| 09-complete-write-path-deep-analysis.md | 3 项 | ✅ 已验证 |
| README-analysis-summary.md | 3 项 | ✅ 已验证 |

### 剩余轻微偏差 (无需修正)
| 文档 | 偏差项 | 偏差范围 | 处理建议 |
|------|-------|---------|---------|
| 00-analysis-overview.md | 统计数据 | 代码变动导致 | 标注代码版本 |
| 02-write-path-overview.md | 统计数据 | 代码变动导致 | 标注代码版本 |
| 03-query-path-overview.md | 统计数据 | 代码变动导致 | 标注代码版本 |
| 06-index-architecture.md | 统计数据 | 代码变动导致 | 标注代码版本 |

---

## 建议后续操作

### 必须执行 (P0)
- [x] 修正文档 09 的 3 个严重错误
- [x] 修正文档 14 的引用错误

### 建议执行 (P1)
- [ ] 在所有文档开头添加代码版本标识 (commit hash/tag)
- [ ] 更新文档 00、02、03、06 的统计数据
- [ ] 建立文档与代码的同步机制 (CI 检查)

### 可选优化 (P2)
- [ ] 为关键函数添加"最后验证日期"标注
- [ ] 建立文档行号自动校验脚本
- [ ] 将验证报告纳入 CI/CD 流程

---

## 修正前后对比

### 关键引用修正
| 函数/结构 | 修正前行号 | 修正后行号 | 偏差 |
|-----------|-----------|-----------|------|
| shard.WriteRows | 468 | 512 | 44 行 |
| MemTable | 57 | 297 | 240 行 |
| WAL.Write | 218 | 248 | 30 行 |

### 函数签名修正
| 函数 | 修正前 | 修正后 |
|------|-------|-------|
| WAL.writeBinary | `writeBinary(data []byte, typ WriteWalType, maxTime int64)` | `writeBinary(walRecord *walRecord) error` |
| WAL.Write | 未标注 | `Write(rows []byte, typ WalRecordType, maxRowTime int64) error` |

---

**验证负责人**: AI Assistant
**验证方法**: 逐行阅读源代码 + 调用链追踪
**验证工具**: Read/Grep/Glob/Bash
