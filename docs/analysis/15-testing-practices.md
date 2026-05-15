# openGemini 测试实践与 Failpoint 机制

**代码版本**: 7032447bce3ec6d35eb204a0f317882298b774b5 (2026-05-15)

本文档详细记录 openGemini 的测试框架、Failpoint 机制和测试最佳实践。

---

## 1. Go 测试基础

### 1.1 测试命令格式

```bash
# 标准测试命令格式
go test [flags] [packages]

# openGemini 完整测试命令
go test -tags "failpoint" -failfast -short -v -count 1 -p 1 -timeout 10m \
    -coverprofile coverage_1.txt -coverpkg ./... ./...
```

### 1.2 关键参数说明

| 参数 | 说明 | openGemini 设置 |
|------|------|----------------|
| `-tags` | 构建标签 | `failpoint` (必需) |
| `-failfast` | 首次失败即停止 | 启用 |
| `-short` | 跳过长测试 | 启用 |
| `-v` | 详细输出 | 启用 |
| `-count` | 运行次数 | `1` (禁用缓存) |
| `-p` | 并行包数 | `1` (避免资源竞争) |
| `-timeout` | 超时时间 | `10m` |
| `-coverprofile` | 覆盖率报告文件 | `coverage_1.txt` |
| `-coverpkg` | 覆盖率包范围 | `./...` (全部) |

### 1.3 测试包排除

以下包从测试中排除：

```bash
# 排除 tests 和 lifted 目录
go list ./... | grep -vE "tests|lifted"
```

**排除原因**:
- `tests/`: 集成测试单独运行
- `lifted/`: 第三方 lifted 代码，不修改

---

## 2. Failpoint 机制

### 2.1 什么是 Failpoint

Failpoint 是一种测试注入机制，允许在测试时：
- 模拟错误和异常
- 注入延迟
- 跳过特定代码路径
- 修改返回值

### 2.2 Failpoint 启用/禁用

```bash
# 启用 failpoint（编译时注入）
make failpoint-enable

# 等价命令
find $PWD/ -type d | grep -vE "(\.git|\.github|\.tests)" | xargs failpoint-ctl enable

# 禁用 failpoint（恢复源码）
make failpoint-disable

# 等价命令
find $PWD/ -type d | grep -vE "(\.git|\.github|\.tests)" | xargs failpoint-ctl disable
```

**注意事项**:
- ✅ 必须在测试前启用
- ✅ 测试后必须禁用（恢复源码）
- ⚠️ 启用后会修改源码（添加 failpoint 代码）
- ⚠️ 禁用后会恢复源码（移除 failpoint 代码）

### 2.3 Failpoint 测试命令

```bash
# 必须包含 -tags "failpoint"
go test -tags "failpoint" -v ./engine/shard

# 单测试函数
go test -tags "failpoint" -run TestWriteRows ./engine/shard

# 带覆盖率
go test -tags "failpoint" -coverprofile coverage.txt ./engine/...
```

### 2.4 Failpoint 代码示例

```go
// 源码中的 failpoint 注入点
import "github.com/pingcap/failpoint"

func (s *Shard) WriteRows(rows []Row) error {
    // 注入点：模拟写入失败
    failpoint.Inject("write-rows-error", func(val failpoint.Value) {
        if val.(bool) {
            failpoint.Return(errors.New("injected error"))
        }
    })
    
    // 正常逻辑
    // ...
}
```

**测试中使用**:

```go
func TestWriteRowsError(t *testing.T) {
    // 启用 failpoint
    failpoint.Enable("write-rows-error", "return(true)")
    defer failpoint.Disable("write-rows-error")
    
    // 执行测试
    err := shard.WriteRows(rows)
    assert.Error(t, err)
    assert.Equal(t, "injected error", err.Error())
}
```

---

## 3. 测试类型

### 3.1 单元测试

```bash
# 运行所有单元测试
make gotest

# 单个包
go test -tags "failpoint" -v ./engine/shard

# 单个测试函数
go test -tags "failpoint" -run TestWriteRows ./engine/shard

# 单个测试函数（带详细输出）
go test -tags "failpoint" -run TestWriteRows -v ./engine/shard
```

### 3.2 集成测试

```bash
# 前置条件
make start-subscriber
make go-build
bash scripts/install_cluster.sh

# 运行集成测试
make integration-test

# 等价命令
URL=http://127.0.0.1:8086 \
  go test -mod=mod -test.parallel 1 -timeout 10m ./tests -v GOCACHE=off

# 清理
make stop-subscriber
```

### 3.3 测试分类

| 类型 | 位置 | 前置条件 | 超时 |
|------|------|----------|------|
| 单元测试 | `*/..._test.go` | 无 | 10m |
| 集成测试 | `tests/integration/` | 运行中的集群 | 10m |
| 功能测试 | `tests/functional/` | 无 | 10m |

---

## 4. 测试覆盖率

### 4.1 生成覆盖率报告

```bash
# 运行测试并生成报告
go test -tags "failpoint" -coverprofile coverage.txt ./...

# 查看覆盖率
go tool cover -func coverage.txt

# 生成 HTML 报告
go tool cover -html coverage.txt -o coverage.html
```

### 4.2 覆盖率文件

CI 中生成的覆盖率文件：

```
coverage_1.txt  # 第一个测试包
coverage_2.txt  # 第二个测试包
...
```

**上传到 Codecov**:
```yaml
# .github/workflows/ci.yml
- name: Upload coverage to Codecov
  uses: codecov/codecov-action@v3
  with:
    token: ${{ secrets.CODECOV_TOKEN }}
```

### 4.3 覆盖率排除

以下代码不计算覆盖率：
- `tests/`: 测试代码本身
- `lifted/`: 第三方 lifted 代码
- 测试文件：`*_test.go`

---

## 5. 测试最佳实践

### 5.1 测试命名规范

```go
// 格式：Test<Type><Function>
func TestShardWriteRows(t *testing.T)      // 测试 Shard.WriteRows
func TestMemTableInsert(t *testing.T)     // 测试 MemTable.Insert
func TestWALWriteBinary(t *testing.T)     // 测试 WAL.writeBinary

// 表格测试
func TestWriteRowsMultiple(t *testing.T) {
    tests := []struct {
        name  string
        input []Row
        want  error
    }{
        {"empty rows", []Row{}, nil},
        {"single row", []Row{{}}, nil},
        {"multiple rows", []Row{{}, {}}, nil},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // 测试逻辑
        })
    }
}
```

### 5.2 测试清理

```go
func TestSomething(t *testing.T) {
    // 1. 准备资源
    resource := setupResource(t)
    
    // 2. 注册清理（defer 确保执行）
    defer cleanupResource(t, resource)
    
    // 3. 执行测试
    testLogic(t, resource)
}
```

### 5.3 错误处理

```go
func TestWriteRows(t *testing.T) {
    // ❌ 错误：忽略错误
    shard.WriteRows(rows)
    
    // ✅ 正确：检查错误
    err := shard.WriteRows(rows)
    if err != nil {
        t.Fatalf("WriteRows failed: %v", err)
    }
    
    // ✅ 更好：使用 testify
    assert.NoError(t, err)
}
```

### 5.4 并发测试

```go
func TestConcurrentWrite(t *testing.T) {
    var wg sync.WaitGroup
    numGoroutines := 10
    
    for i := 0; i < numGoroutines; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            // 并发写入逻辑
            shard.WriteRows([]Row{{ID: id}})
        }(i)
    }
    
    wg.Wait()
}
```

---

## 6. 测试超时处理

### 6.1 默认超时

- **单元测试**: 10 分钟
- **集成测试**: 10 分钟
- **CI 限制**: 30 分钟

### 6.2 调整超时

```bash
# 增加超时时间
go test -timeout 30m ./...

# 减少超时时间（快速失败）
go test -timeout 1m ./engine/shard
```

### 6.3 长测试标记

```go
// 标记为长测试（会被 -short 跳过）
func TestLongRunning(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping long-running test")
    }
    
    // 长测试逻辑
}
```

---

## 7. 测试工具

### 7.1 测试框架

| 工具 | 用途 | 版本 |
|------|------|------|
| `testing` | Go 标准测试框架 | 内置 |
| `testify` | 断言库 | v1.10.0 |
| `goconvey` | BDD 风格测试 | v2.11.0 |
| `failpoint` | 测试注入 | pingcap/failpoint |

### 7.2 安装测试工具

```bash
# testify (断言库)
go get github.com/stretchr/testify

# goconvey (BDD 框架)
go get github.com/smartystreets/goconvey

# failpoint (测试注入)
go get github.com/pingcap/failpoint
go install github.com/pingcap/failpoint/failpoint-ctl@latest
```

---

## 8. 调试测试

### 8.1 运行单个测试

```bash
# 运行匹配的测试
go test -tags "failpoint" -run TestName ./package/path

# 正则匹配
go test -tags "failpoint" -run "TestWrite|TestRead" ./engine/...

# 子测试
go test -tags "failpoint" -run "TestWriteRows/empty_rows" ./engine/shard
```

### 8.2  verbose 输出

```bash
# 详细输出（显示所有 log）
go test -tags "failpoint" -v ./...

# 仅失败时输出
go test -tags "failpoint" -v -short ./... 2>&1 | grep -A 5 "FAIL"
```

### 8.3 调试技巧

```go
func TestDebug(t *testing.T) {
    // 1. 打印日志
    t.Logf("Debug info: %v", value)
    
    // 2. 跳过测试
    t.Skip("reason for skipping")
    
    // 3. 标记失败（继续执行）
    t.Error("non-fatal error")
    
    // 4. 标记失败（停止执行）
    t.Fatal("fatal error")
}
```

---

## 9. 常见问题

### 9.1 测试因缺少 failpoint 失败

```bash
# 错误信息
# undefined: failpoint

# 解决：添加 -tags "failpoint"
go test -tags "failpoint" -v ./...
```

### 9.2 Failpoint 启用后无法编译

```bash
# 问题：failpoint 注入后编译失败
# 解决：禁用 failpoint 并重新启用
make failpoint-disable
make failpoint-enable
go test -tags "failpoint" -v ./...
```

### 9.3 测试超时

```bash
# 问题：panic: test timed out after 10m0s

# 解决 1：增加超时
go test -timeout 30m ./...

# 解决 2：查找慢测试
go test -tags "failpoint" -v -json ./... | grep -A 5 "Slowest"
```

### 9.4 覆盖率报告未生成

```bash
# 问题：coverage.txt 为空

# 解决：确保 -coverprofile 和 -coverpkg 正确
go test -tags "failpoint" -coverprofile coverage.txt -coverpkg ./... ./...
```

### 9.5 集成测试失败

```bash
# 前置条件检查清单
# 1. Subscriber 已启动
make start-subscriber

# 2. 二进制文件已构建
make go-build

# 3. 集群已安装
bash scripts/install_cluster.sh

# 4. 进程正在运行
ps -ef | grep ts-

# 5. 运行测试
make integration-test

# 6. 清理
make stop-subscriber
```

---

## 10. CI 中的测试

### 10.1 GitHub Actions 配置

```yaml
# .github/workflows/ci.yml
jobs:
  UT:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.24"
      - name: Run gotest
        run: |
          go mod tidy
          make gotest
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
```

### 10.2 测试执行顺序

CI 中测试执行顺序：

```mermaid
graph LR
    A[Checkout] --> B[Setup Go]
    B --> C[go mod tidy]
    C --> D[make gotest]
    D --> E[Upload Codecov]
```

---

## 11. 相关文档

- `00-analysis-plan.md` - 分析计划
- `02-storage-engine.md` - 存储引擎（含测试）
- `08-code-review-report.md` - 代码评审报告
- `14-build-ci-guide.md` - 构建与 CI/CD 指南
- `../Makefile` - Makefile 测试目标
- `../Makefile.common` - 公共测试配置
- `../.github/workflows/ci.yml` - CI 配置

---

## 12. 快速参考

### 12.1 常用测试命令

```bash
# 启用 failpoint
make failpoint-enable

# 运行所有测试
make gotest

# 运行单个包
go test -tags "failpoint" -v ./engine/shard

# 运行单个测试
go test -tags "failpoint" -run TestWriteRows ./engine/shard

# 生成覆盖率
go test -tags "failpoint" -coverprofile coverage.txt ./...

# 查看覆盖率
go tool cover -func coverage.txt

# 禁用 failpoint
make failpoint-disable

# 运行集成测试
make integration-test
```

### 12.2 测试检查清单

- [ ] Failpoint 已启用
- [ ] 环境变量已设置（GO111MODULE, GONOSUMDB, GOSUMDB, TZ）
- [ ] CGO 标志已设置
- [ ] 测试超时合理（默认 10m）
- [ ] 覆盖率报告生成
- [ ] 集成测试前置条件满足

---

**创建日期**: 2026-05-15  
**最后更新**: 2026-05-15  
**最后验证**: 2026-05-15  
**验证状态**: ✅ 新建  
**代码版本**: 7032447bce3ec6d35eb204a0f317882298b774b5
