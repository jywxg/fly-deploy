这是一个非常专业且优雅的工程实践！将硬编码的名称提取为环境变量，可以让这份 GitHub Actions 工作流变成一个通用的“部署模板”。以后你在 GitHub 上创建新的仓库来部署其他项目时，只需要把这个文件复制过去，改一下顶部的几个变量名就能直接用了。

为了实现这个目标，我们在工作流的顶部统一使用 `env` 声明全局环境变量，然后在执行命令时通过 `${{ env.变量名 }}` 的语法进行引用。我还顺便把“硬盘大小”也提取成了变量，方便你以后部署需要大容量硬盘的项目。

以下是修改后的最终完整版本，没有任何删减或省略：

```yaml
name: Deploy to Fly.io

on:
  push:
    branches:
      - main # 当有代码推送到 main 分支时触发自动部署
  workflow_dispatch: # 添加手动触发按钮，方便在网页上随时直接运行

# 在这里集中管理你的自定义变量，修改此处即可应用到整个部署流程
env:
  APP_NAME: "vaultwarden-webdav"  # 自定义应用名称
  REGION: "sin"                   # 自定义部署节点地区 (例如: sin, nrt, hkg)
  VOLUME_NAME: "vw_data"          # 自定义持久化存储卷名称
  VOLUME_SIZE: "1"                # 自定义存储卷大小 (单位：GB)

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

### 💡 搭配使用的重要提醒

当你把这份工作流作为模板用到下一个全新的项目时，请记得同步检查你的 `fly.toml` 文件：

因为 Fly.io 的核心是“基础设施即代码”，`fly deploy` 命令在最终执行部署时，依然会读取你代码仓库中的 `fly.toml` 文件。因此，当你修改了上面工作流中的 `APP_NAME`、`REGION` 或 `VOLUME_NAME` 时，请务必确保 `fly.toml` 文件中的以下三处配置也改成了相同的名字，否则部署时可能会产生冲突：

1. 第一行的应用名：`app = "你的新应用名"`
2. 第二行的地区：`primary_region = "你的新地区"`
3. 底部的存储卷挂载源：`source = "你的新硬盘名"`

现在你的这套架构已经完全实现了自动化、模板化和零成本待机。如果这套工作流成功运行亮起了绿灯，需要我为你梳理一下后续如何通过网页或客户端连接你的密码库吗？
