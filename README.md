# 心流直觉｜华科大全球校友创新创业大赛路演

本仓库保存可继续编辑的 HTML 路演源文件，并通过 GitHub Pages 提供在线演示。

## 在线演示

GitHub Pages 启用后访问：

https://yt-x-beijing.github.io/flow-intuition-hust-innovation/

## 目录结构

```text
.
├── index.html                         # 唯一演示源文件，也是 Pages 入口
├── assets/                            # 原始图片资源，供后续编辑使用
├── references/
│   ├── LIBERO10_EXTERNAL_MEMORY_DEMO.md
│   └── P4_TECHNICAL_LOGIC.md          # P4 技术架构的权威约束
├── .github/workflows/pages.yml        # GitHub Pages 自动部署
├── AGENTS.md                          # Coding Agent 修改规范
└── .nojekyll
```

## 本地查看

直接双击 `index.html`，或在仓库目录运行：

```bash
python3 -m http.server 8000
```

然后访问 `http://localhost:8000/`。

## 使用 Coding Agent 修改

1. Clone 本仓库并在 Coding Agent 中打开仓库目录。
2. 先阅读 `AGENTS.md` 和相关 `references/` 文件。
3. 直接修改 `index.html`；它是唯一演示源文件。
4. 在 1920×1080、1280×720 和手机视口下检查页面。
5. 提交并推送到 `main`，GitHub Pages 会自动更新。

常用命令：

```bash
git clone https://github.com/YT-X-Beijing/flow-intuition-hust-innovation.git
cd flow-intuition-hust-innovation
python3 -m http.server 8000
```

## 演示操作

- 方向键、空格键：翻页
- 手机端：左右滑动
- `E`：进入或退出文字编辑模式
- `Ctrl/Cmd + S`：保存浏览器内编辑内容

浏览器内编辑保存在当前浏览器本地。需要同步给其他协作者的修改，必须写回 `index.html` 并提交到 GitHub。
