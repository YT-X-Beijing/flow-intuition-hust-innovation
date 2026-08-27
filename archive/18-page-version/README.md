# 18页完整路演版本

本目录保存压缩为 5 分钟版本之前的完整 18 页 HTML 路演稿，其中包含 P1—P17 和附录 A1。

## 文件

- `index.html`：18 页完整版本，可直接在浏览器打开，也可复制到独立分支继续修改。

## 与主版本的关系

- 仓库根目录 `index.html`：当前 12 页主版本，也是 GitHub Pages 默认入口。
- 本目录 `index.html`：18 页历史版本，不参与 Pages 默认部署。
- Git tag `archive-18-page-v1`：指向 12 页压缩前的原始完整提交，可精确恢复当时的全部仓库状态。

## 恢复建议

如需基于 18 页版本继续开发，优先从 tag 创建新分支：

```bash
git switch -c restore-18-page archive-18-page-v1
```

这样不会覆盖当前 12 页主版本。
