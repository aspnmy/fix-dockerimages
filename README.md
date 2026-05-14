# fix-dockerimages

一键自动化修补 Docker 基础镜像，并自动推送到远端镜像仓库的 Shell 工具集。

---

## 项目简介

`fix-dockerimages` 是一套轻量级 Shell 脚本工具，用于：

- 批量拉取 Docker 基础镜像
- 自动修补/加固/定制镜像
- 更新并推送到远端镜像仓库

适用于企业内部基础镜像标准化、安全加固、预装组件、权限规范和配置固化场景。

---

## 核心特性

- 自动拉取并处理基础镜像
- 镜像安全加固与权限优化
- 组件预装与环境定制
- 自动清理冗余文件、缩小镜像体积
- 支持多仓库凭证管理（DockerHub / GHCR / DHI）
- 一键清理扫描报告
- 支持“离开模式”：安全清空环境变量
- 可选支持 s6-overlay 容器化运行

---

## 目录结构

```text
fix-dockerimages/
├── bin/             # 工具脚本目录（secure-env 等）
├── config/          # 配置文件目录
├── auto-fiximage    # 主执行脚本
├── .gitignore
└── readme.md
```

---

## 快速开始

### 1. 克隆项目

```bash
git clone https://github.com/aspnmy/fix-dockerimages.git
cd fix-dockerimages
```

### 2. 配置用户目标仓库和镜像凭证

项目根目录下的 `.user_repo` 文件用于指定修补后镜像的目标仓库地址，例如：

```text
aspnmy/debian-base
```

如果需要推送到私有仓库，还可在 `bin/.env` 中添加如下环境变量：

```bash
DOCKERHUB_USER=xxx
DOCKERHUB_PASS=xxx
GHCR_USER=xxx
GHCR_PASS=xxx
DHI_USER=xxx
DHI_PASS=xxx
```

> `.user_repo` 是必须文件，内容应为目标镜像仓库路径；如果没有配置凭证，工具仍可在本地执行修补，但推送操作可能无法完成。

### 3. 执行镜像修补

```bash
./auto-fiximage "docker镜像远端路径"
```

如果未提供镜像路径，则脚本可能无法正确执行，请务必传入有效的镜像地址。

---

## 常用命令

```bash
# 运行镜像修补流程
./auto-fiximage "docker镜像远端路径"

# 清理扫描报告
./bin/secure-env cls

# 离开模式：清空密钥并锁定环境
./bin/secure-env leave
```

---

## 依赖环境

- Docker
- Bash
- s6-overlay（可选，用于容器化运行）

---

## 应用场景

- 企业内部基础镜像标准化
- 镜像安全漏洞批量修复
- SSH、组件、权限统一固化
- CI/CD 流水线基础镜像自动更新

---

