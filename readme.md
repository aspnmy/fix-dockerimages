# Fix Docker Images Shell Tool: Automated Patch and Push

# fix\-dockerimages

一键自动化修补 Docker 基础镜像，并自动推送到远端镜像仓库的 Shell 工具集。

---

## 项目介绍

fix\-dockerimages 是一套轻量 Shell 脚本工具，用于**批量拉取 Docker 基础镜像 → 自动修补 / 加固 / 定制 → 推送至远端仓库**。
适合企业内部统一 Base 镜像、安全加固、预装组件、权限规范、配置固化等场景。

---

## 功能特性

- 自动拉取基础镜像

- 内置镜像安全加固、权限优化、组件预装逻辑

- 自动清理冗余文件、缩小镜像体积

- 支持多仓库（DockerHub/GHCR/DHI）凭证管理

- 一键清理扫描报告

- 支持离开模式（安全清空环境变量）

- 基于 s6\-overlay 稳定运行（可选）

---

## 目录结构

```Plain Text
fix-dockerimages/
├── bin/             # 工具脚本目录（secure-env 等）
├── config/          # 配置文件目录
├── auto-fiximage    # 主执行脚本
├── .gitignore
└── readme.md
```

---

## 快速开始

### 1\. 克隆项目

```bash
git clone https://github.com/aspnmy/fix-dockerimages.git
cd fix-dockerimages
```

### 2\. 配置镜像凭证（可选）

在 `bin/\.env` 填写镜像仓库账号密码：

```bash
DOCKERHUB_USER=xxx
DOCKERHUB_PASS=xxx
GHCR_USER=xxx
GHCR_PASS=xxx
DHI_USER=xxx
DHI_PASS=xxx
```

### 3\. 执行修补

```bash
./auto-fiximage "docker镜像远端路径"
```
如果不带入 docker镜像远端路径 则执行代码中的默认地址

---

## 常用命令

```bash
# 正常执行镜像修补
./auto-fiximage "docker镜像远端路径"

# 清理扫描报告
./bin/secure-env cls

# 离开模式：清空密钥并锁定环境
./bin/secure-env leave
```

---

## 环境依赖

- Docker

- Bash

- s6\-overlay（可选，用于容器化运行）

---

## 使用场景

- 企业内部基础镜像标准化

- 镜像安全漏洞批量修复

- SSH、组件、权限统一固化

- CI/CD 流水线基础镜像自动更新

---

