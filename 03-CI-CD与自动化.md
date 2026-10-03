
> **解决什么问题**：
> 	每次改代码都要手动构建、测试、部署，又慢又容易出错。

> **一句话**：
> 	把"写代码 → 上线"的重复步骤自动化，
> 	让**代码一推送就自动构建测试部署**。

## 1. CI / CD 各自的概念

| 术语     | 全称                                        | 做什么                | 阶段      |
| ------ | ----------------------------------------- | ------------------ | ------- |
| **CI** | Continuous Integration（持续集成）              | 代码合并后自动**构建 + 测试** | 合并到主分支时 |
| **CD** | Continuous Delivery / Deployment（持续交付/部署） | 自动**部署**到环境        | CI 通过后  |

```
开发 push 代码
    ↓
CI：自动编译 + 跑测试（发现问题马上知道）✅
    ↓
CD：通过后自动部署到 测试/预发/生产
    ↓
线上新版本就绪
```

> 用一句话记：**CI 管"任务会不会爆"，CD 管"代码怎么上线"**。

> CI 让每个提交都被验证，CD 让验证过的代码自动上生产。

## 2. 一条典型流水线（Pipeline）的步骤

```
① 触发     push 代码 / 合并 MR
② 检出     checkout 代码
③ 装依赖    install（npm / pip / maven...）
④ 构建     build（编译、打包、build 镜像）
⑤ 测试     test（单测、lint、静态检查）
⑥ 镜像/产物  docker build & push
⑦ 部署     deploy 到 测试→预发→生产
⑧ 通知     成功/失败 通知到 IM/邮件
```

> 某一步失败 → 流水线停，通知开发者。
> **问题在最早期暴露**，是 CI/CD 最大价值。

## 3. 主流工具对比

| 工具                    | 形态           | 配置方式                      | 特点                |
| --------------------- | ------------ | ------------------------- | ----------------- |
| **GitHub Actions**    | 云端，GitHub 内置 | `.github/workflows/*.yml` | 最流行，和 GitHub 深度集成 |
| **GitLab CI**         | 云端/自托管       | `.gitlab-ci.yml`          | GitLab 生态         |
| **Jenkins**           | 自托管          | 脚本/UI                     | 老牌、灵活、要自己运维       |
| **CircleCI / Travis** | 云端           | config 文件                 | 老牌 SaaS           |

> 原理都差不多：**机器上按配置执行一串命令**，只是托管位置和配置形式不同。

## 4. GitHub Actions 实战

```yaml
# .github/workflows/deploy.yml

name: CI
on:
  push:
    branches: [main]        # 触发条件：push 到 main
jobs:
  build-test-deploy:
    runs-on: ubuntu-latest  # 跑在什么机器上
    steps:
      - uses: actions/checkout@v4        # ① 检出代码
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - name: Install
        run: pip install -r requirements.txt   # ③ 装依赖
      - name: Test
        run: pytest                           # ⑤ 测试
      - name: Build & Push Docker
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/me/api:latest          # ⑥ 镜像推送
      - name: Deploy
        run: ssh deploy@server "docker pull ... && docker compose up -d"  # ⑦ 部署
        
```

> 核心语法：
> `on`（触发）→ `jobs`（并行任务）→ `steps`（串行步骤，每步一个命令/action）。

## 5. 环境划分（重要）

| 环境                | 用途      | 何时部署            |
| ----------------- | ------- | --------------- |
| **开发 dev**        | 本地/开发联调 | 随便              |
| **测试/预发 staging** | 接近生产地验证 | CI 通过后          |
| **生产 prod**       | 用户真正用的  | 预发验证后，通常人工确认或定时 |

> CD 常分两步：**自动部署到 staging，生产要"一键确认"再上**（安全性考虑）。

## 6. 高频一句话

- **CI = 自动构建+测试**，**CD = 自动部署**，合起来是"push 即上线"
- **流水线 = 触发 → 检出 → 装依赖 → 构建 → 测试 → 打包 → 部署 → 通知**
- **失败提前暴露**是核心价值，别等上线才发现
- GitHub Actions 最常用，配置文件在 `.github/workflows/`
- **生产环境通常不自动无脑部署**，预发过了再人工放行