# BGP 路由协议排错指南 · 精读笔记

《Troubleshooting BGP》(Vinit Jain / Brad Edgeworth, Cisco Press) 中文版逐章精读笔记。

**在线阅读**：https://huyushao521.github.io/bgp-book-notes/

左侧目录按「部分 → 章」分组，可折叠；每页有跟随滚动高亮的本页目录。

## 目录结构

```
_config.yml                          站点配置
_layouts/default.html                页面布局（左侧目录 + 正文）
assets/css/site.css                  样式
index.md                             首页
chapters/chapter-01-bgp-fundamentals.md   第 1 章
```

## 新增一章

1. 在 `chapters/` 下新建 `chapter-NN-xxx.md`，直接用 Markdown 写，无需 front matter
2. 编辑 `_layouts/default.html`，把该章对应的 `<li class="pending">` 换成链接：

```html
<li><a href="{{ '/chapters/chapter-NN-xxx.html' | relative_url }}"
       data-path="chapters/chapter-NN-xxx.html">第 N 章 · 标题</a></li>
```

3. 更新 `index.md` 的目录表，然后提交推送
