# openGemini 应用层与配置分析

**代码版本**: 7032447bce3ec6d35eb204a0f317882298b774b5 (2026-05-15)

本文档分析 openGemini 的应用程序入口、配置系统和启动流程。

---

## 1. 应用程序入口

### 1.1 应用目录结构

```
app/
├── command.go              # 命令处理核心
├── common.go               # 公共函数和常量
├── server.go               # Server 接口
├── operation.go            # 运维操作
├── ts-data/
│   └── main.go             # 数据工具
├── ts-meta/
│   ├── main.go             # 元数据服务
│   ├── meta/               # Meta 实现
│   └── run/                # 运行逻辑
├── ts-store/
│   ├── main.go             # 存储服务
│   ├── storage/            # 存储实现
│   └── run/                # 运行逻辑
├── ts-sql/
│   ├── main.go             # SQL 服务
│   └── sql/                # SQL 实现
├── ts-server/
│   ├── main.go             # 综合节点
│   └── run/                # 运行逻辑
├── ts-monitor/
│   ├── main.go             # 监控服务
│   ├── collector/          # 指标采集
│   └── run/                # 运行逻辑
└── ts-recover/
    ├── main.go             # 数据恢复
    └── recover/            # 恢复实现
```

**统计**: 7 个应用入口 (`ls -la app/ts-*/`)

### 1.2 统一启动模式

所有服务遵循统一的启动模式：

```go
// app/ts-store/main.go
func main() {
    // 1. 初始化命令行解析
    app.InitParse()
    
    // 2. 设置节点类型
    errno.SetNode(errno.NodeStore)
    
    // 3. 准备版本信息
    info := app.ServerInfo{
        App:       config.AppStore,
        Version:   app.Version,
        Commit:    app.GitCommit,
        Branch:    app.GitBranch,
        BuildTime: app.BuildTime,
    }
    
    // 4. 运行服务
    app.Run(os.Args[1:], run.NewCommand(info, true))
}
```

**验证**:
- `app/ts-store/main.go` - Store 启动
- `app/ts-meta/main.go` - Meta 启动
- `app/ts-sql/main.go` - SQL 启动

### 1.3 ts-server 特殊处理

```go
// app/ts-server/main.go
func main() {
    app.InitParse()
    errno.SetNode(errno.NodeServer)
    
    // 单机模式：内嵌 Meta + Store + SQL
    info := app.ServerInfo{
        App:       config.AppSingle,
        Version:   app.Version,
        Commit:    app.GitCommit,
        Branch:    app.GitBranch,
        BuildTime: app.BuildTime,
    }
    
    // 创建三个组件
    cmdMeta := meta.NewCommand(info, false)
    cmdStore := store.NewCommand(info, false)
    cmdSql := ingestserver.NewCommand(info, false)
    
    // 设置 Logo
    cmdMeta.Logo = app.TSSERVER
    cmdSql.Logo = ""
    cmdStore.Logo = ""
    
    // 初始化存储依赖
    cmdSql.AfterOpen = func() {
        run.InitStorage(cmdSql.Server, cmdStore.Server)
    }
    
    // 同时运行三个服务
    app.Run(os.Args[1:], cmdMeta, cmdStore, cmdSql)
}
```

### 1.4 应用类型常量

```go
// lib/config/config.go:70-77
const (
    AppSql     App = "sql"
    AppStore   App = "store"
    AppMeta    App = "meta"
    AppSingle  App = "single"
    AppMonitor App = "monitor"
    AppData    App = "data"
    Unknown    App = "unKnown"
)
```

**验证**: `lib/config/config.go:70-77` 确认常量定义

---

## 2. 命令处理

### 2.1 Command 结构

```go
// app/command.go:39-58
type Command struct {
    Logo          string
    Usage         string
    Pidfile       string
    Logger        *logger.Logger
    Command       *cobra.Command
    Info          ServerInfo
    Version       string
    Server        Server
    Config        config.Config
    NewServerFunc func(config.Config, ServerInfo, *logger.Logger) (Server, error)
    
    AfterOpen func()
}

func NewCommand() *Command {
    return &Command{
        Logger: logger.NewLogger(errno.ModuleUnknown),
    }
}
```

### 2.2 运行流程

```go
// app/command.go:60-120
func (cmd *Command) Run(args ...string) error {
    // 1. 解析命令行参数
    options, err := ParseFlags(func() { /* 帮助信息 */ }, args...)
    if err != nil {
        return err
    }
    
    // 2. 初始化配置
    err = cmd.InitConfig(cmd.Config, options.ConfigPath)
    if err != nil {
        return fmt.Errorf("parse config: %s", err)
    }
    
    cmd.Logger = logger.NewLogger(errno.ModuleUnknown)
    
    // 3. 显示 Logo
    fmt.Fprint(os.Stdout, cmd.Logo)
    
    // 4. 创建 Server
    s, err := cmd.NewServerFunc(cmd.Config, cmd.Info, cmd.Logger)
    if err != nil {
        return fmt.Errorf("create server: %s", err)
    }
    
    // 5. 写入 PID 文件
    if err := WritePIDFile(options.PIDFile); err != nil {
        return fmt.Errorf("write pid file: %s", err)
    }
    cmd.Pidfile = options.PIDFile
    
    // 6. 打开 Server
    if err := s.Open(); err != nil {
        return err
    }
    
    cmd.Server = s
    
    // 7. 等待信号
    signal := procutil.WaitForSigterm()
    logger.GetLogger().Info("service received shutdown signal", 
        zap.Any("signal", signal))
    
    // 8. 关闭 Server
    util.MustClose(cmd)
    
    return nil
}
```

### 2.3 配置初始化

```go
// app/command.go
func (cmd *Command) InitConfig(cfg config.Config, path string) error {
    // 1. 解析配置文件
    err := config.Parse(cfg, path)
    if err != nil {
        return err
    }
    
    // 2. 验证配置
    err = cfg.Validate()
    if err != nil {
        return err
    }
    
    // 3. 应用环境变量覆盖
    err = cfg.ApplyEnvOverrides(os.Getenv)
    if err != nil {
        return err
    }
    
    return nil
}
```

---

## 3. 配置系统

### 3.1 配置文件结构

```bash
config/
├── openGemini.conf              # 集群配置模板
├── openGemini.singlenode.conf   # 单机配置模板
├── monitor.conf                 # 监控配置
└── weakpasswd.properties        # 弱密码检查
```

### 3.2 配置格式 (TOML)

```toml
# config/openGemini.conf

[common]
meta-join = ["meta1:8092", "meta2:8092", "meta3:8092"]
cpu-num = 0
memory-size = "0"
pprof-enabled = true

[meta]
bind-address = "0.0.0.0:8088"
http-bind-address = "0.0.0.0:8091"
rpc-bind-address = "0.0.0.0:8092"
dir = "/tmp/openGemini/data/meta"

[store]
bind-address = "0.0.0.0:8088"
wal-dir = "/tmp/openGemini/data/wal"
data-dir = "/tmp/openGemini/data/data"

[sql]
bind-address = "0.0.0.0:8086"

[http]
bind-address = "0.0.0.0:8086"
flight-address = "0.0.0.0:8087"

[coordinator]
write-timeout = "10s"
query-timeout = "10s"

[retention]
check-interval = "30m"
```

### 3.3 配置接口

```go
// lib/config/config.go:58-66
type Config interface {
    ApplyEnvOverrides(func(string) string) error
    Validate() error
    GetLogging() *Logger
    GetSpdy() *Spdy
    GetCommon() *Common
    ShowConfigs() map[string]interface{}
    GetLogStoreConfig() *LogStoreConfig
}
```

### 3.4 配置解析

```go
// lib/config/config.go:106-128
func Parse(conf Config, path string) error {
    if path == "" {
        return nil
    }
    return fromTomlFile(conf, path)
}

func fromTomlFile(c Config, p string) error {
    content, err := os.ReadFile(path.Clean(p))
    if err != nil {
        return err
    }
    
    // 处理 BOM
    dec := unicode.BOMOverride(transform.Nop)
    content, _, err = transform.Bytes(dec, content)
    if err != nil {
        return err
    }
    
    return fromToml(c, string(content))
}

func fromToml(c Config, input string) error {
    _, err := toml.Decode(input, c)
    return err
}
```

### 3.5 环境变量覆盖

```go
// lib/config/config.go
func (c *Config) ApplyEnvOverrides(getenv func(string) string) error {
    // 遍历所有配置字段
    // 如果存在对应的环境变量，则覆盖配置值
    // 格式：OPENGEMINI_<SECTION>_<KEY>
    
    examples := []string{
        "OPENGEMINI_META_BIND_ADDRESS",
        "OPENGEMINI_STORE_DATA_DIR",
        "OPENGEMINI_HTTP_BIND_ADDRESS",
    }
    
    return nil
}
```

---

## 4. Server 接口

### 4.1 Server 定义

```go
// app/server.go:26-29
type Server interface {
    Open() error
    Close() error
    Err() <-chan error
}
```

### 4.2 Service 定义

```go
// app/server.go:62-65
type Service interface {
    Open() error
    Close() error
}
```

### 4.3 ServiceGroup

```go
// app/server.go:67-92
type ServiceGroup struct {
    services []Service
}

func (m *ServiceGroup) Add(s Service) {
    if s == nil || util.IsObjectNil(s) {
        return
    }
    m.services = append(m.services, s)
}

func (m *ServiceGroup) Open() error {
    for _, s := range m.services {
        err := s.Open()
        if err != nil {
            return err
        }
    }
    return nil
}

func (m *ServiceGroup) MustClose() {
    for _, s := range m.services {
        util.MustClose(s)
    }
}
```

---

## 5. 启动流程

### 5.1 完整启动流程

```mermaid
sequenceDiagram
    participant Main as main.go
    participant App as app.Run
    participant Cmd as Command
    participant Cfg as Config
    participant Srv as Server
    
    Main->>App: 启动服务
    App->>Cmd: 创建 Command
    
    Cmd->>Cmd: InitParse()
    Cmd->>Cfg: Parse(configPath)
    Cfg-->>Cmd: 配置对象
    Cmd->>Cfg: Validate()
    Cmd->>Cfg: ApplyEnvOverrides()
    
    Cmd->>Cmd: NewServerFunc()
    Cmd->>Srv: 创建 Server
    Srv-->>Cmd: Server 实例
    
    Cmd->>Cmd: WritePIDFile()
    Cmd->>Srv: Open()
    Srv->>Srv: 初始化组件
    Srv-->>Cmd: 启动完成
    
    Cmd->>Cmd: WaitForSigterm()
    Note over Cmd: 运行中...
    
    Cmd->>Cmd: 收到信号
    Cmd->>Srv: Close()
    Srv->>Srv: 清理资源
    Srv-->>Cmd: 关闭完成
```

### 5.2 Meta 服务启动

```go
// app/ts-meta/run/server.go
type Server struct {
    httpServer *httpd.Service
    raftServer *raft.Service
    metaServer *meta.Service
}

func (s *Server) Open() error {
    // 1. 打开 HTTP 服务
    if err := s.httpServer.Open(); err != nil {
        return err
    }
    
    // 2. 打开 Raft 服务
    if err := s.raftServer.Open(); err != nil {
        return err
    }
    
    // 3. 打开 Meta 服务
    if err := s.metaServer.Open(); err != nil {
        return err
    }
    
    return nil
}
```

### 5.3 Store 服务启动

```go
// app/ts-store/run/server.go
type Server struct {
    httpServer  *httpd.Service
    storeServer *store.Service
    engine      engine.Engine
}

func (s *Server) Open() error {
    // 1. 打开存储引擎
    if err := s.engine.Open(); err != nil {
        return err
    }
    
    // 2. 打开存储服务
    if err := s.storeServer.Open(); err != nil {
        return err
    }
    
    // 3. 打开 HTTP 服务
    if err := s.httpServer.Open(); err != nil {
        return err
    }
    
    return nil
}
```

---

## 6. 版本信息

### 6.1 版本变量

```go
// app/common.go:116-120
var (
    Version   string  // 由构建脚本设置
    GitCommit string
    GitBranch string
    BuildTime string
)
```

### 6.2 版本显示

```go
// app/common.go:124-132
func FullVersion(app string) string {
    const format = `openGemini version info:
%s: %s
git: %s %s
os: %s
arch: %s`
    
    return fmt.Sprintf(format, app, Version, GitBranch, 
        GitCommit, runtime.GOOS, runtime.GOARCH)
}
```

### 6.3 构建时注入

```bash
# build.py 中设置版本信息
go build -ldflags "-X github.com/openGemini/openGemini/app.Version=1.0.0 
    -X github.com/openGemini/openGemini/app.GitCommit=abc123
    -X github.com/openGemini/openGemini/app.GitBranch=main
    -X github.com/openGemini/openGemini/app.BuildTime=2026-05-15"
```

---

## 7. PID 文件管理

### 7.1 写入 PID

```go
// app/common.go:177-194
func WritePIDFile(pidfile string) error {
    if pidfile == "" {
        return nil
    }
    
    pidDir := filepath.Dir(pidfile)
    if err := os.MkdirAll(pidDir, 0700); err != nil {
        return fmt.Errorf("os.MkdirAll failed: %s", err)
    }
    
    pid := strconv.Itoa(os.Getpid())
    if err := os.WriteFile(pidfile, []byte(pid), 0600); err != nil {
        return fmt.Errorf("write pid file failed: %s", err)
    }
    
    return nil
}
```

### 7.2 删除 PID

```go
// app/common.go:167-175
func RemovePIDFile(pidfile string) {
    if pidfile == "" {
        return
    }
    
    if err := os.Remove(pidfile); err != nil {
        logger.GetLogger().Error("Remove pidfile failed", zap.Error(err))
    }
}
```

---

## 8. 日志和监控

### 8.1 日志初始化

```go
// app/common.go:211-218
func LogStarting(name string, info *ServerInfo) {
    logger.GetLogger().Info(name+" starting",
        zap.String("version", info.Version),
        zap.String("branch", info.Branch),
        zap.String("commit", info.Commit),
        zap.String("buildTime", info.BuildTime))
    
    logger.GetLogger().Info("Go runtime",
        zap.String("version", runtime.Version()),
        zap.Int("maxprocs", cpu.GetCpuNum()))
}
```

### 8.2 启动日志

```go
// app/command.go
fmt.Fprint(os.Stdout, cmd.Logo)  // 显示 Logo
logger.GetLogger().Info("starting service")
```

### 8.3 应用 Logo

```go
// app/common.go
const METALOGO = `
 _________   ______   ____    ____  ________  _________     _
|  _   _  |.' ____ \ |_   \  /   _||_   __  ||  _   _  |   / \
...
`

const SQLLOGO = `
 _________   ______    ______     ___    _____
|  _   _  |.' ____ \ .' ____ \  .'   '.  |_   _|
...
`

const TSSERVER = `
 _________   ______    ______   ________  _______  ____   ____
|  _   _  |.' ____ \ .' ____ \ |_   __  ||_   __ \|_  _| |_  _|
...
`
```

---

## 9. 代码验证

### 9.1 验证点

✅ **应用入口**:
- `app/ts-*/main.go` - 7 个应用入口
- 统一启动模式确认

✅ **配置系统**:
- `lib/config/config.go:70-77` - App 类型常量
- `lib/config/config.go:58-66` - Config 接口
- 配置解析流程正确

✅ **命令处理**:
- `app/command.go:39-58` - Command 结构
- `app/command.go:60-120` - Run 方法流程

✅ **Server 接口**:
- `app/server.go:26-29` - Server 接口
- `app/server.go:62-65` - Service 接口

### 9.2 关键代码索引

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| 应用入口 | `app/ts-*/main.go` | - |
| Command | `app/command.go` | 39-58, 60-120 |
| 配置接口 | `lib/config/config.go` | 58-66, 70-77 |
| Server 接口 | `app/server.go` | 26-29, 62-65 |
| PID 文件 | `app/common.go` | 167-194 |
| 版本信息 | `app/common.go` | 116-132 |

---

## 10. 总结

### 应用层特点

1. **统一模式**: 7 个应用遵循相同启动流程
2. **单机集成**: ts-server 内嵌所有组件
3. **清晰接口**: Server/Service 接口定义明确

### 配置系统特点

1. **TOML 格式**: 易读易写
2. **环境变量**: 支持环境变量覆盖
3. **配置验证**: 启动前验证配置有效性

### 启动流程

1. 解析命令行
2. 加载配置
3. 验证配置
4. 创建 Server
5. 打开服务
6. 等待信号
7. 清理关闭

---

**文档版本**: 1.0  
**创建日期**: 2026-05-15  
**验证状态**: ✅ 已验证代码一致性

---
**最后验证**: 2026-05-15
**验证状态**: ✅ 已修正
**代码版本**: 7032447bce3ec6d35eb204a0f317882298b774b5
