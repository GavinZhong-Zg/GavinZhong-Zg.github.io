# GitHub Actions 工作流方案对比与选择指

## 🔧 两种主流工作流方案对比

| 特性维度 | **方案A：社区方案 (`peaceiris/actions-gh-pages`)** | **方案B：官方方案 (`actions/deploy-pages`)** |
| :--- | :--- | :--- |
| **核心原理** | 使用 Git 将生成的 `public` 文件夹推送到指定的分支（如 `gh-pages` 或 `main`）。 | 将 `public` 文件夹打包为“产物”，通过 GitHub Pages 专用环境部署。 |
| **成熟度** | **极高**，在 Hexo、Jekyll 等静态生成器社区中经过多年广泛验证。 | **较新**，由 GitHub 官方推出，代表了未来的部署方向。 |
| **配置复杂度** | 相对简单、集中，易于理解和调试。 | 稍复杂，需要拆分为 `build` 和 `deploy` 两个任务，并配置权限和环境。 |
| **可靠性** | 久经考验，社区资源丰富，遇到问题极易搜索到解决方案。 | 依赖 GitHub 官方服务，但针对 Hexo 的具体实践案例相对较少。 |
| **关键命令** | 明确使用 `npx hexo generate`。 | 依赖 `npm run build`（需在 `package.json` 中额外配置）。 |
| **主要风险点** | 无明显风险。 | **`npm run build` 命令未定义**是导致失败的最常见原因。 |

## 📝 方案B（官方方案）的配置要点与隐患

你提供的配置代码本身是有效的，但**必须进行一项关键修改**，否则必定会失败。

### 必须完成的修改：
在你的 Hexo 项目根目录下的 `package.json` 文件中，确保 `scripts` 部分包含 `build` 命令：

```json
{
  "scripts": {
    "build": "hexo generate", // 这是关键！将 build 命令指向 hexo generate
    "clean": "hexo clean",
    "server": "hexo server"
  }
}
```
如果缺少这步，工作流运行到 `npm run build` 时会直接报错，因为默认的 Hexo 项目并没有预定义 `build` 脚本。

### 方案B的潜在优势：
- **官方集成**：与 GitHub Pages 服务深度集成，可能享受更优的发布管道。
- **产物管理**：使用“产物（artifact）”概念，理论上在多次构建间管理文件更清晰。

## 🎯 最终建议：如何选择？

### 推荐使用 **方案A (`peaceiris/actions-gh-pages`)**，原因如下：
1.  **专注稳定**：它是为静态博客（尤其是 Hexo）量身定制的方案，经过了海量用户和项目的验证。
2.  **心智负担小**：流程直观（`生成 -> 推送`），易于理解和排查问题。
3.  **社区支持强**：你遇到的几乎任何错误，都能在网上找到完全匹配的解决方案。

### 可以考虑 **方案B (`actions/deploy-pages`)** 的场景：
- 你希望尝试 GitHub 最新的官方部署特性。
- 你的项目需要非常复杂的、多阶段的构建流程，官方方案可能提供更精细的控制。

## 🚀 操作指南（基于推荐的方案A）

1.  **创建工作流文件**：
    在项目根目录创建 `.github/workflows/deploy.yml`，内容如下：

    ```yaml
    name: Deploy Hexo Site

    on:
      push:
        branches: [ main ] # 或 master

    jobs:
      build-and-deploy:
        runs-on: ubuntu-latest
        steps:
          - name: Checkout
            uses: actions/checkout@v4
            with:
              fetch-depth: 0

          - name: Setup Node.js
            uses: actions/setup-node@v4
            with:
              node-version: '20'
              cache: 'npm'

          - name: Install Dependencies
            run: npm ci

          - name: Generate
            run: npx hexo generate

          - name: Deploy
            uses: peaceiris/actions-gh-pages@v4
            with:
              github_token: ${{ secrets.GITHUB_TOKEN }}
              # 用户站点仓库 (username.github.io) 用 main:
              publish_branch: main
              # 项目站点仓库 (其他名称) 用 gh-pages:
              # publish_branch: gh-pages
              publish_dir: ./public
    ```

2.  **提交并推送**：
    ```bash
    git add .github/workflows/deploy.yml
    git commit -m "添加自动化部署工作流"
    git push origin main
    ```

3.  **在仓库设置中启用 Pages**：
    前往仓库的 `Settings > Pages`，将 **Source** 设置为 **GitHub Actions**。

## ⚠️ 注意事项
- 无论选择哪种方案，请再次确认 Hexo 根目录 `_config.yml` 中的 `url` 已正确设置为 `https://gavinzhong-zg.github.io`。
- 首次运行 Actions 可能需要几分钟时间，请耐心等待并在 **Actions** 标签页查看实时日志。
- 如果选择方案B，请务必、务必、务必确认 `package.json` 中已配置 `build` 脚本。

**总结**：对于你的 Hexo 个人博客，从**稳定、省心、易排查**的角度出发，**方案A (`peaceiris/actions-gh-pages`) 是更优选择**。