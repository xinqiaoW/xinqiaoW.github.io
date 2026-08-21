# Repository instructions

## Jekyll 博客 Markdown 兼容规则

编辑或新增 `_posts/*.md` 时，按 Jekyll/Kramdown 网页语法处理，不要直接保留 Obsidian 专用语法。

- 高亮：使用 `<mark>文字</mark>`，不要在正文中使用 Obsidian 的 `==文字==`。
- 标准链接：使用 `[显示文字](https://example.com)`，不要使用 `[[页面]]`、`[[URL]]` 或多层嵌套链接。
- 标签链接：使用 `[显示文字](/tags#FrontMatterTag)`，其中锚点必须与文章 front matter 中的标签完全一致。
- 图片：使用 `![替代文字](/images/.../image.jpg)`，并确保资源路径能从站点根目录访问。
- 缩写标题：USM、FUM、AI 等需要保持大写时，在 front matter 中显式设置 `title`，不要只依赖文件名自动生成标题。
- 检查范围：查找 `[[` 和正文中的 `==` 时，不要修改 fenced code block 内合法的比较运算符或源码。
- 发布前：确认 Obsidian 语法已清理、链接目标存在、图片加载成功，并检查 GitHub Pages 构建及线上文章。

示例：

```markdown
交易 Hash <mark>0xabc...</mark>
[经济模型](/tags#EconomicModelFault)
[Etherscan 合约源码](https://etherscan.io/address/0x...)
```
