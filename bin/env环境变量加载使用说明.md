## 目录整体结构

```Plain Text
docker-image-fix-scan/
├── bin/
│   └── secure-env          # 独立安全环境变量工具
├── auto-fiximage      # 主业务脚本
├── 环境变量使用说明.md      # 完整帮助文档
└── .env.template           # .env模板（改名即可用）
```

---

## 1\. bin/secure\-env （已优化：无\.env 只告警、不中断主脚本）

```bash
#!/bin/bash
# 安全环境变量加载工具
# 调用方式：source ./bin/secure-env
# 返回值：
# 0 = 成功加载.env并销毁
# 1 = 无.env文件，沿用系统环境变量

load_env_securely() {
    local env_file=".env"

    if [ ! -f "${env_file}" ]; then
        echo "ℹ️  提示：未检测到 .env 敏感配置文件"
        echo "ℹ️  将沿用当前系统已有环境变量，脚本继续运行..."
        return 1
    fi

    echo "🔐 检测到 .env 配置文件，开始安全加载环境变量..."
    # 批量导出所有变量到当前Shell
    set -o allexport
    source "${env_file}"
    set +o allexport

    # 加载后立即销毁敏感文件
    rm -f "${env_file}"
    echo "✅ 环境变量已导入内存 → .env 文件已安全销毁"
    return 0
}

# 自动执行加载逻辑
load_env_securely
```

授可执行权限：

```bash
chmod +x bin/secure-env
```

---

## 2\. 主脚本：镜像自动修复扫描\.sh

```bash
#!/bin/bash
set -euo pipefail

# ==========================
# 引入独立安全环境变量工具
# 无.env仅告警，不中断主体流程
# ==========================
source ./bin/secure-env

# ==========================
# 仓库登录变量检查（空值则交互式手动输入）
# ==========================
check_and_read_credits() {
    # Docker Hub
    if [ -z "${DOCKERHUB_USER:-}" ] || [ -z "${DOCKERHUB_PASS:-}" ]; then
        echo -e "\n⚠️ Docker Hub 凭证为空，请手动输入："
        read -p "用户名：" DOCKERHUB_USER
        read -s -p "密码：" DOCKERHUB_PASS
        echo
    fi

    # GHCR.IO
    if [ -z "${GHCR_USER:-}" ] || [ -z "${GHCR_PASS:-}" ]; then
        echo -e "\n⚠️ GHCR.IO 凭证为空，请手动输入："
        read -p "用户名：" GHCR_USER
        read -s -p "密码/Token：" GHCR_PASS
        echo
    fi

    # DHI.IO
    if [ -z "${DHI_USER:-}" ] || [ -z "${DHI_PASS:-}" ]; then
        echo -e "\n⚠️ DHI.IO 凭证为空，请手动输入："
        read -p "用户名：" DHI_USER
        read -s -p "密码：" DHI_PASS
        echo
    fi
}

# 执行凭证校验
check_and_read_credits

# ==========================
# 镜像配置 支持命令行传参
# ==========================
if [ $# -ge 1 ]; then
    IMAGE_NAME="$1"
else
    IMAGE_NAME="rdamssh/dhi-debian-base:bookworm-debian12-dev"
fi

IMAGE_REPO_NAME="${IMAGE_NAME%:*}"
IMAGE_TAG="${IMAGE_NAME#*:}"

# ==========================
# 三大仓库常量
# ==========================
DH="docker.io"
GHCR="ghcr.io"
DHI="dhi.io"

# ==========================
# 修复后镜像标签
# ==========================
FIXED_TAG="${IMAGE_TAG}-fixed"
FIXED_IMAGE="${IMAGE_REPO_NAME}:${FIXED_TAG}"

# ==========================
# 登录全部仓库 复用内存环境变量
# ==========================
login_all_registry() {
    echo -e "\n=== 登录 Docker Hub ==="
    echo "$DOCKERHUB_PASS" | docker login $DH -u "$DOCKERHUB_USER" --password-stdin

    echo -e "\n=== 登录 GHCR.IO ==="
    echo "$GHCR_PASS" | docker login $GHCR -u "$GHCR_USER" --password-stdin

    echo -e "\n=== 登录 DHI.IO ==="
    echo "$DHI_PASS" | docker login $DHI -u "$DHI_USER" --password-stdin

    echo -e "✅ 所有仓库登录完成！"
}

# ==========================
# 业务流程函数
# ==========================
pull_original() {
    echo -e "\n=== 拉取原始镜像 ==="
    docker pull "$IMAGE_NAME"
}

build_fixed_image() {
    echo -e "\n=== 构建系统漏洞修复镜像 ==="
    cat <<EOF > fix-baseimages.Dockerfile
FROM ${IMAGE_NAME}
RUN apt-get clean \
    && apt-get update -y \
    && apt-get upgrade -y \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
EOF
    docker build -f fix-baseimages.Dockerfile -t "$FIXED_IMAGE" .
}

tag_all() {
    echo -e "\n=== 为修复镜像打三大仓库标签 ==="
    docker tag "$FIXED_IMAGE" "${DH}/${FIXED_IMAGE}"
    docker tag "$FIXED_IMAGE" "${GHCR}/${FIXED_IMAGE}"
    docker tag "$FIXED_IMAGE" "${DHI}/${FIXED_IMAGE}"
}

scan_fixed() {
    echo -e "\n=== 离线漏洞扫描修复后镜像 ==="
    mkdir -p scan_report
    REPORT_FILE="scan_report/scan-$(date +%Y%m%d%H%M%S)-${FIXED_TAG}.md"

    trivy image \
        --scanners vuln \
        --skip-db-update \
        --offline-scan \
        --format github \
        "$FIXED_IMAGE" > "$REPORT_FILE"

    echo "✅ 扫描报告已生成：$REPORT_FILE"
}

# ==========================
# 主入口
# ==========================
main() {
    login_all_registry
    pull_original
    build_fixed_image
    tag_all
    scan_fixed

    echo -e "\n🎉 全流程执行完成！"
    echo "原始镜像：$IMAGE_NAME"
    echo "修复镜像：$FIXED_IMAGE"
    echo "已打标签："
    echo "  ${DH}/${FIXED_IMAGE}"
    echo "  ${GHCR}/${FIXED_IMAGE}"
    echo "  ${DHI}/${FIXED_IMAGE}"
}

main
```

授可执行权限：

```bash
chmod +x auto-fiximage
```

---

## 3\. \.env\.template 模板文件

```env
# DockerHub 仓库账号密码
DOCKERHUB_USER=
DOCKERHUB_PASS=

# GHCR.IO 仓库账号Token
GHCR_USER=
GHCR_PASS=

# DHI 私有仓库账号密码
DHI_USER=
DHI_PASS=
```

使用时复制为正式配置：

```bash
cp .env.template .env
# 填入账号密码即可
```

---

## 4\. 环境变量使用说明\.md

```markdown
# 环境变量使用说明
## 一、功能概述
1. 本地临时创建 `.env` 存放三大镜像仓库账号密钥
2. 通过 `bin/secure-env` 工具加载到系统Shell环境变量
3. 加载完成**自动删除 .env**，密钥仅留存内存，不落地磁盘
4. 无 `.env` 文件时仅提示、**不中断主脚本运行**
5. 环境变量为空时自动交互式手动输入

## 二、目录结构
```

docker\-image\-fix\-scan/
├── bin/
│   └── secure\-env          \# 安全环境变量加载工具
├── 镜像自动修复扫描\.sh      \# 主执行脚本
├── 环境变量使用说明\.md      \# 本文档
└── \.env\.template           \# 密钥配置模板

```Plain Text
## 三、配置使用步骤
1. 复制模板为正式配置：
   ```bash
   cp .env.template .env
```

2. 编辑 `\.env` 填入各仓库用户名、密码 / Token

3. 运行主脚本，工具自动加载并销毁配置文件

## 四、工具调用方式

### 1\. 脚本内引用

主脚本固定头部引入：

```bash
source ./bin/secure-env
```

### 2\. 手动重新加载密钥

修改 `\.env` 后重载：

```bash
source ./bin/secure-env
```

## 五、返回值逻辑

- 返回 0：成功加载 `\.env` 并安全销毁

- 返回 1：无 `\.env` 文件，沿用系统环境变量，**不影响主流程**

## 六、脚本内变量调用

直接使用内存环境变量：

```bash
${DOCKERHUB_USER}
${DOCKERHUB_PASS}
${GHCR_USER}
${GHCR_PASS}
${DHI_USER}
${DHI_PASS}
```

## 七、安全规范

1. 严禁将 `\.env` 提交 Git、网盘、截图外泄

2. 仅本地临时使用，加载后自动销毁

3. 退出终端自动清空内存环境变量

4. 变量缺失自动弹窗输入，无硬编码密钥

```Plain Text
---

## 整体交付使用流程
1. 解压整个 `docker-image-fix-scan` 目录
2. 执行授权：
   ```bash
   chmod +x bin/secure-env auto-fiximage
```

3. 复制模板配置：`cp \.env\.template \.env` 填密钥

4. 运行：

    ```bash
    # 默认镜像
    ./auto-fiximage
    # 自定义传参镜像
    ./auto-fiximage rdamssh/dhi-debian-base:bookworm-fips-dev
    ```
