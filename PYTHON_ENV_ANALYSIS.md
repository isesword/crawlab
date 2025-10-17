# Crawlab Python 环境管理分析

## 概述

Crawlab 是一个基于 Golang 的分布式爬虫管理平台，它支持多种编程语言和框架，包括 Python、Node.js、Go 等。本文档详细分析 Crawlab 如何管理 Python 环境。

---

## 1. Python 环境架构

### 1.1 基础架构

Crawlab 采用 **pyenv** 作为 Python 版本管理工具，通过以下方式管理 Python 环境：

```
系统层级:
├── Docker 基础镜像 (Ubuntu 24.04)
│   ├── pyenv 安装脚本
│   ├── Python 3.12 (默认版本)
│   └── 基础依赖库
│
└── 任务运行时
    ├── pyenv 环境变量配置
    ├── PATH 路径设置
    └── 依赖自动安装 (可选)
```

---

## 2. Python 安装和管理

### 2.1 Docker 基础镜像配置

文件: `/workspaces/crawlab/docker/base-image/Dockerfile`

```dockerfile
FROM ubuntu:24.04

# 安装 Python
RUN bash /app/install/python/python.sh install 3.12
```

### 2.2 Python 安装脚本

文件: `/workspaces/crawlab/docker/base-image/install/python/python.sh`

这是一个功能完整的 Python 版本管理脚本，主要功能包括：

#### 核心功能

1. **setup_pyenv()**: 安装和配置 pyenv
   ```bash
   # 安装 pyenv
   curl https://pyenv.run | bash
   
   # 创建环境变量文件
   export PYENV_ROOT="$HOME/.pyenv"
   export PATH="$PYENV_ROOT/bin:$PATH"
   eval "$(pyenv init -)"
   eval "$(pyenv virtualenv-init -)"
   ```

2. **install_dependencies()**: 安装 Python 编译所需的系统依赖
   ```bash
   apt install -y \
     make build-essential \
     libssl-dev zlib1g-dev \
     libxml2-dev libxslt-dev \
     libffi-dev libbz2-dev \
     libreadline-dev libsqlite3-dev \
     xz-utils liblzma-dev
   ```

3. **install [version]**: 安装指定版本的 Python
   ```bash
   bash python.sh install 3.12
   ```

4. **handle_requirements()**: 处理依赖安装
   - 支持自定义 requirements 内容
   - 回退到默认的 `requirements.txt`

5. **create_symlinks()**: 创建全局符号链接
   ```bash
   ln -sf $(pyenv which python) /usr/local/bin/python
   ln -sf $(pyenv which python3) /usr/local/bin/python3
   ln -sf $(pyenv which pip) /usr/local/bin/pip
   ```

### 2.3 默认安装的 Python 包

文件: `/workspaces/crawlab/docker/base-image/install/python/requirements.txt`

```plaintext
crawlab-sdk>=0.7.0rc5    # Crawlab SDK
scrapy                    # 爬虫框架
selenium                  # 浏览器自动化
bs4                       # BeautifulSoup4
requests                  # HTTP 库
```

---

## 3. 任务执行时的 Python 环境配置

### 3.1 环境变量配置

文件: `/workspaces/crawlab/core/task/handler/runner_config.go`

#### 核心函数: `configurePythonPath()`

```go
func (r *Runner) configurePythonPath() {
    // 1. 获取 pyenv 根路径
    pyenvRoot := utils.GetPyenvPath()  // 默认: /root/.pyenv
    pyenvShimsPath := pyenvRoot + "/shims"
    pyenvBinPath := pyenvRoot + "/bin"
    
    // 2. 设置 PYENV_ROOT 环境变量
    r.cmd.Env = append(r.cmd.Env, "PYENV_ROOT="+pyenvRoot)
    
    // 3. 更新 PATH
    currentPath := r.getEnvFromCmd("PATH")
    if currentPath == "" {
        currentPath = os.Getenv("PATH")
    }
    newPath := pyenvBinPath + ":" + pyenvShimsPath + ":" + currentPath
    r.setEnvInCmd("PATH", newPath)
}
```

#### PATH 优先级顺序

```
/root/.pyenv/bin -> /root/.pyenv/shims -> 系统原有 PATH
```

这确保了 pyenv 管理的 Python 版本优先于系统自带的 Python。

### 3.2 pyenv 路径配置

文件: `/workspaces/crawlab/core/utils/config.go`

```go
const DefaultPyenvPath = "/root/.pyenv"

func GetPyenvPath() string {
    // 支持通过配置文件自定义
    if res := viper.GetString("install.pyenv.path"); res != "" {
        return res
    }
    return DefaultPyenvPath
}
```

可以通过配置文件 `config.yml` 自定义 pyenv 路径：
```yaml
install:
  pyenv:
    path: /custom/path/.pyenv
```

---

## 4. 自动依赖安装机制

### 4.1 配置选项

Spider 模型中包含 `AutoInstall` 字段：

文件: `/workspaces/crawlab/core/models/models/spider.go`

```go
type Spider struct {
    // ... 其他字段
    AutoInstall bool `json:"auto_install" bson:"auto_install"`
    // ...
}

func (s *Spider) GetAutoInstall() (autoInstall bool) {
    return s.AutoInstall
}
```

### 4.2 依赖安装接口

文件: `/workspaces/crawlab/core/interfaces/dependency_installer_service.go`

```go
type DependencyInstallerService interface {
    // 检查是否启用自动安装
    IsAutoInstallEnabled() (enabled bool)
    
    // 根据 Spider ID 获取安装依赖的命令
    GetInstallDependencyRequirementsCmdBySpiderId(id primitive.ObjectID) (cmd *exec.Cmd, err error)
}
```

### 4.3 支持的依赖文件

根据 CHANGELOG 信息，Crawlab 支持从以下文件自动安装依赖：

- **Python**: `requirements.txt`
- **Node.js**: `package.json`

#### 安装过程（推测）

1. 检查爬虫目录是否存在 `requirements.txt`
2. 如果存在且 `AutoInstall=true`，执行：
   ```bash
   pip install -r requirements.txt
   ```

---

## 5. 完整的任务执行环境配置

### 5.1 环境变量设置流程

文件: `/workspaces/crawlab/core/task/handler/runner_config.go`

```go
func (r *Runner) configureEnv() {
    // 1. 初始化基础环境变量
    r.cmd.Env = os.Environ()
    
    // 2. 配置 Python 路径
    r.configurePythonPath()
    
    // 3. 配置 Node.js 路径
    r.configureNodePath()
    
    // 4. 配置 Go 路径
    r.configureGoPath()
    
    // 5. 清除旧的 CRAWLAB_ 前缀环境变量
    for i := 0; i < len(r.cmd.Env); i++ {
        env := r.cmd.Env[i]
        if strings.HasPrefix(env, "CRAWLAB_") {
            r.cmd.Env = append(r.cmd.Env[:i], r.cmd.Env[i+1:]...)
            i--
        }
    }
    
    // 6. 添加任务特定环境变量
    r.cmd.Env = append(r.cmd.Env, "CRAWLAB_TASK_ID="+r.tid.Hex())
    
    // 7. 添加全局环境变量
    envs, err := client.NewModelService[models.Environment]().GetMany(nil, nil)
    for _, env := range envs {
        r.cmd.Env = append(r.cmd.Env, env.Key+"="+env.Value)
    }
    
    // 8. 添加父进程 PID（用于子进程识别）
    r.cmd.Env = append(r.cmd.Env, "CRAWLAB_PARENT_PID="+fmt.Sprint(os.Getpid()))
}
```

### 5.2 工作目录配置

```go
func (r *Runner) configureCwd() {
    workspacePath := utils.GetWorkspace()
    if r.s.GitId.IsZero() {
        // 非 Git 项目
        r.cwd = filepath.Join(workspacePath, r.s.Id.Hex())
    } else {
        // Git 项目
        r.cwd = filepath.Join(workspacePath, r.s.GitId.Hex(), r.s.GitRootPath)
    }
}
```

---

## 6. Python 环境管理的最佳实践

### 6.1 版本管理

#### 默认版本
- Crawlab 默认安装 **Python 3.12**
- 通过 pyenv 管理，可以安装多个版本

#### 安装其他版本
```bash
# 进入容器
docker exec -it crawlab_master bash

# 列出可用版本
bash /app/install/python/python.sh list

# 安装新版本
bash /app/install/python/python.sh install 3.11

# 设置全局版本
pyenv global 3.11
```

### 6.2 依赖管理

#### 方式 1: 使用 requirements.txt（推荐）

在爬虫项目根目录创建 `requirements.txt`：
```plaintext
scrapy==2.5.0
requests==2.28.0
beautifulsoup4==4.11.0
```

启用自动安装：
```json
{
  "auto_install": true
}
```

#### 方式 2: 在基础镜像中预装

修改 `/workspaces/crawlab/docker/base-image/install/python/requirements.txt`：
```plaintext
crawlab-sdk>=0.7.0rc5
scrapy
selenium
bs4
requests
pandas           # 新增
numpy            # 新增
```

重新构建镜像：
```bash
cd /workspaces/crawlab/docker/base-image
docker build -t crawlabteam/crawlab-base:latest .
```

#### 方式 3: 通过 Web UI 管理依赖

根据 CHANGELOG，Crawlab 0.4.3+ 版本支持通过 Web 界面安装/卸载依赖。

### 6.3 虚拟环境

虽然 Crawlab 使用 pyenv 管理 Python 版本，但每个任务都在独立的进程中运行，共享全局 Python 环境。

如果需要为不同爬虫使用不同的依赖版本，建议：
1. 使用 Docker 部署多个 Worker 节点
2. 每个节点配置不同的 Python 环境
3. 通过节点标签分配任务

---

## 7. 依赖同步机制

### 7.1 gRPC 依赖同步服务

文件: `/workspaces/crawlab/core/grpc/server/dependencies_server_v2.go`

```go
func (svr DependenciesServerV2) Sync(ctx context.Context, request *grpc.DependenciesServiceV2SyncRequest) {
    // 1. 获取节点信息
    n, _ := service.NewModelServiceV2[models2.NodeV2]().GetOne(
        bson.M{"key": request.NodeKey}, nil)
    
    // 2. 获取数据库中的依赖记录
    depsDb, _ := service.NewModelServiceV2[models2.DependencyV2]().GetMany(
        bson.M{
            "node_id": n.Id,
            "type":    request.Lang,  // "python", "node", etc.
        }, nil)
    
    // 3. 同步依赖列表
    // - 添加新依赖
    // - 删除不存在的依赖
}
```

### 7.2 依赖数据模型

文件: `/workspaces/crawlab/core/models/models/v2/dependency_v2.go`

```go
type DependencyV2 struct {
    Name          string             `json:"name" bson:"name"`
    NodeId        primitive.ObjectID `json:"node_id" bson:"node_id"`
    Type          string             `json:"type" bson:"type"`        // "python", "node"
    Version       string             `json:"version" bson:"version"`
    LatestVersion string             `json:"latest_version"`
}
```

---

## 8. 调试和故障排查

### 8.1 查看 Python 环境

```bash
# 进入容器
docker exec -it crawlab_master bash

# 查看当前 Python 版本
python --version

# 查看 pyenv 安装的所有版本
pyenv versions

# 查看 Python 路径
which python
which pip

# 查看环境变量
echo $PYENV_ROOT
echo $PATH
```

### 8.2 查看已安装的包

```bash
# 列出所有已安装的包
pip list

# 查看特定包的信息
pip show scrapy
```

### 8.3 任务执行日志

任务执行时，可以在日志中查看：
- Python 版本
- 环境变量
- 依赖安装过程
- 错误信息

---

## 9. 配置参数总结

### 9.1 环境变量

| 变量名 | 说明 | 默认值 |
|--------|------|--------|
| `PYENV_ROOT` | pyenv 根目录 | `/root/.pyenv` |
| `PATH` | 执行路径 | `$PYENV_ROOT/bin:$PYENV_ROOT/shims:...` |
| `CRAWLAB_TASK_ID` | 任务 ID | 运行时生成 |
| `CRAWLAB_GRPC_ADDRESS` | gRPC 服务地址 | 配置文件指定 |
| `CRAWLAB_PARENT_PID` | 父进程 PID | 运行时生成 |

### 9.2 配置文件选项

文件: `config.yml`

```yaml
install:
  pyenv:
    path: /root/.pyenv  # pyenv 安装路径
  node:
    path: /usr/lib/node_modules  # Node.js 模块路径
    bin: /usr/local/bin/node     # Node.js 可执行文件路径
```

---

## 10. 架构优势

### 10.1 灵活性
- 支持多版本 Python 共存
- 可以通过 pyenv 轻松切换版本
- 支持自定义安装路径

### 10.2 隔离性
- 每个任务在独立进程中运行
- 环境变量隔离
- 支持节点级别的环境差异化

### 10.3 可扩展性
- 支持自动依赖安装
- 支持通过 Web UI 管理依赖
- 支持多节点分布式部署

### 10.4 易维护性
- 集中化的脚本管理
- 标准化的安装流程
- 完善的日志和监控

---

## 11. 未来改进建议

1. **虚拟环境支持**: 为每个爬虫创建独立的虚拟环境
2. **依赖版本锁定**: 支持 `requirements.lock` 文件
3. **conda 支持**: 作为 pyenv 的补充选项
4. **依赖缓存**: 加速依赖安装过程
5. **依赖分析**: 检测依赖冲突和安全问题

---

## 12. 参考资料

- [pyenv 官方文档](https://github.com/pyenv/pyenv)
- [Crawlab 官方文档](https://docs.crawlab.cn)
- [Python 虚拟环境最佳实践](https://docs.python.org/3/tutorial/venv.html)
- Crawlab 源码仓库相关文件：
  - `docker/base-image/install/python/python.sh`
  - `core/task/handler/runner_config.go`
  - `core/utils/config.go`
  - `core/interfaces/dependency_installer_service.go`

---

## 总结

Crawlab 的 Python 环境管理采用了现代化的工具链和架构设计：

1. **版本管理**: 使用 pyenv 管理多个 Python 版本
2. **依赖管理**: 支持自动安装 requirements.txt 中的依赖
3. **环境隔离**: 每个任务在独立进程中运行，环境变量隔离
4. **灵活配置**: 支持通过配置文件自定义各种路径和选项
5. **分布式支持**: 支持多节点部署，每个节点可以有不同的环境配置

这种设计既保证了灵活性和可扩展性，又简化了部署和维护过程，非常适合企业级爬虫管理平台的需求。
