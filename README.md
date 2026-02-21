# Vaultwarden on Fly.io 自动化部署指南 (零成本待机方案)

本教程旨在指导你如何利用 GitHub Actions，将 Vaultwarden (轻量级的 Bitwarden 密码服务器) 全自动部署到 Fly.io。

本方案的核心优势：

1. **极致省钱（利用豁免机制）**：通过配置 `min_machines_running = 0` 实现按需唤醒（Serverless 模式）。日常待机零计算费用，仅产生极少的存储费用（1GB 约 $0.15/月），完美契合 Fly.io 每月账单低于 $5.00 即自动豁免的“隐形免费”政策。
2. **彻底告别本地命令行**：利用 GitHub Actions 实现云端“建仓+开盘+部署”一条龙，无需在本地安装 Fly CLI。
3. **高安全性与高复用性**：所有敏感信息与部署变量均通过 GitHub Secrets 注入，代码仓库干净且可作为通用模板。

---

## 📋 前置准备

1. 注册并激活 [Fly.io](https://fly.io/) 账号（需绑定有效信用卡以激活 Pay-As-You-Go 计费，但不超额不会扣费）。
2. 准备一个空的 GitHub 仓库。

---

## 🛠️ 第一步：获取 Fly.io 部署令牌 (API Token)

1. 登录 Fly.io 控制台。
2. 在左侧菜单找到 **Account** -> **Tokens**。
3. 点击 **Create Token** 生成一个新的部署令牌。
4. **复制并妥善保存该 Token**（它只显示一次）。

---

## 🔒 第二步：配置 GitHub 仓库机密 (Secrets)

前往你的 GitHub 仓库 -> **Settings** -> **Secrets and variables** -> **Actions** -> **New repository secret**。

依次添加以下 5 个核心配置变量：

| Name (变量名) | Secret (变量值) | 说明 |
| --- | --- | --- |
| `FLY_API_TOKEN` | `你的 Fly.io Token` | 第一步获取的授权密钥 |
| `APP_NAME` | `vaultwarden-webdav` | 你的应用在全球的唯一名称（请自行修改确保不重复） |
| `REGION` | `sin` | 部署节点代码（如 `sin` 新加坡, `nrt` 东京, `hkg` 香港） |
| `VOLUME_NAME` | `vw_data` | 持久化存储卷名称 |
| `VOLUME_SIZE` | `1` | 存储卷大小（单位：GB。1GB 存密码绰绰有余） |

---

## 📄 第三步：编写应用配置文件 (`fly.toml`)

在你的 GitHub 仓库根目录新建名为 `fly.toml` 的文件。我们将管理员密码等敏感信息剔除，仅保留基础环境配置。

请复制以下**最终完整版本**，不要做任何删减：

```toml
app = "vaultwarden-webdav"
primary_region = "sin"

[build]
image = "ghcr.io/workerspages/vaultwarden-webdav:latest"

[env]
DASHBOARD_ADMIN_USER = "admin"
SIGNUPS_ALLOWED = "false"
TZ = "Asia/Shanghai"
WEBSOCKET_ENABLED = "true"

[http_service]
auto_start_machines = true
auto_stop_machines = true
force_https = true
internal_port = 80
min_machines_running = 0
processes = [ "app" ]

[[mounts]]
destination = "/data"
source = "vw_data"

[[vm]]
cpus = 1
memory = "256mb"
memory_mb = 256

```

*(注意：部署时 GitHub Actions 会以这里的 `app`、`primary_region` 和 `source` 为准，请确保它们与你在 GitHub Secrets 中设置的值保持逻辑一致，尽管部署流会优先尝试用 Secrets 创建基础设施。)*

---

## ⚙️ 第四步：配置 GitHub Actions 工作流

在仓库根目录下依次创建文件夹 `.github/workflows/`，并在其中新建文件 `deploy.yml`。

这是一个配置为**纯手动触发**的自动化脚本，它会自动检测并创建缺失的应用和硬盘环境。请复制以下**最终完整版本**：

```yaml
name: Deploy to Fly.io

on:
  workflow_dispatch: # 仅保留手动触发按钮，完全禁止代码推送时的自动部署

# 将所有部署相关的变量从 GitHub Secrets 中安全读取
env:
  APP_NAME: ${{ secrets.APP_NAME }}
  REGION: ${{ secrets.REGION }}
  VOLUME_NAME: ${{ secrets.VOLUME_NAME }}
  VOLUME_SIZE: ${{ secrets.VOLUME_SIZE }}

jobs:
  deploy:
    name: Deploy app
    runs-on: ubuntu-latest
    env:
      FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }} # 全局注入 Fly.io 部署 Token
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Flyctl
        uses: superfly/flyctl-actions/setup-flyctl@master

      - name: Create Fly.io App (if not exists)
        # 尝试使用变量创建应用，如果应用已存在则忽略报错继续往下走
        run: flyctl apps create ${{ env.APP_NAME }} --org personal || true

      - name: Create Persistent Volume (if not exists)
        # 尝试使用变量在指定节点创建存储卷，静默确认 (--yes)，如果已存在则忽略报错
        run: flyctl volumes create ${{ env.VOLUME_NAME }} --app ${{ env.APP_NAME }} --region ${{ env.REGION }} --size ${{ env.VOLUME_SIZE }} --yes || true

      - name: Deploy to Fly.io
        # 执行最终的推送和部署
        run: flyctl deploy --remote-only

```

---

## 🚀 第五步：执行首次部署

1. 确认上述两个文件 (`fly.toml` 和 `.github/workflows/deploy.yml`) 已经提交 (commit) 并推送 (push) 到你的 GitHub 仓库。
2. 在 GitHub 网页端，点击仓库顶部的 **Actions** 标签页。
3. 在左侧菜单点击 **Deploy to Fly.io**。
4. 点击右侧蓝色的 **Run workflow** 按钮，再点击下拉菜单中的 **Run workflow**。
5. 等待进度条走完（出现绿色打勾 ✔️ 标志），你的 Vaultwarden 就已经成功部署并在云端运行了！

---

## 🛡️ 第六步：配置管理员密码 (安全收尾)

为了能够登录 Vaultwarden 的后台管理面板 (`https://你的应用名.fly.dev/admin`)，我们需要通过 Fly.io 的网页端安全地注入管理员密码：

1. 登录 [Fly.io Dashboard](https://fly.io/dashboard)。
2. 进入你刚部署的应用面板。
3. 点击左侧的 **Secrets** 菜单。
4. 点击 **Add Secrets**，添加如下内容：
* **Name**: `DASHBOARD_ADMIN_PASSWORD`
* **Value**: `你自定义的强密码`


5. 保存并生效。Fly.io 会自动重启你的应用，密码配置即刻生效。

**至此，部署全部完成！享受你的专属密码管理器吧！**

