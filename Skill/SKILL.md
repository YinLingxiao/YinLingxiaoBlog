---
name: upload-blog
description: 把博文、Obsidian .md 或 Word 转出的 markdown 整理成墨浅博客可上传的 page bundle，并按 Site API 规则校验。用户说『做成可上传的博客文件夹』『准备上传博客』『整理成 blog bundle』『上传到 blog』『检查博文能不能上传』时使用。只处理博客，不处理笔记。正文润色交给 write-blog。
draft: true
---

# 墨浅博客上传包

产出一个可在 `/blog/admin/upload` 直接选择的文件夹。校验以 `shared/upload/validation.ts`、`Site-api/src/content/policy.ts`、`content-store.ts` 为准。接口 `POST /api/admin/content/blog`。

## 先确定

- 分类：上传页单独填写，不由文件夹决定。常用 `技术`、`随笔`、`阅读`。
- slug：即文件夹名。

## 文件夹

```text
<slug>/
├─ index.md
└─ cover.jpg      # 可选；所有图片与 index.md 同层
```

违反任一条会被拒：

- 只选一个文件夹，内部不能有子文件夹。
- 恰好一个 `index.md`，不能有其他 `.md`。
- 其余文件只能是 `png` `jpg` `jpeg` `gif` `webp` `avif`，且必须是真实图片。
- slug 匹配 `^[a-z0-9]+(?:-[a-z0-9]+)*$`，长度不超过 80。如 `curve-integral`。
- 文件名长度不超过 120，不能有空格和 `% ? # ( ) [ ]`，不能以 `.` 开头或结尾，不能是 `con`/`nul` 等 Windows 保留名。
- `index.md` 为 UTF-8，非空，无 NUL 字节。
- 默认上限：32 个文件；正文 2 MB；单图 10 MB、4000 万像素；合计 50 MB。
- slug 在 `Opus/posts/` 任意分类下不能已存在，已有内容不会被覆盖。
- frontmatter 若写 `category:`，必须与上传时填写的分类完全一致；一般不写。

## index.md

```markdown
---
date: 2026-10-03
summary: 一句话摘要
cover: ./cover.jpg
tags: [标签一, 标签二]
aliases: [别名]
draft: false
---

# 标题

正文……

![图片说明](./figure-1.png)
```

- 标题优先取 `title:`，其次正文第一行 `# H1`（取出后不重复渲染），最后用 slug。
- 想进首页最新 3 篇，必须写合法 `date`（`YYYY-MM-DD`）。`summary` 是卡片摘要。
- `cover` 必须指向同层图片；不写则列表用排版占位，不会拿正文第一张图当封面。
- `tags`、`aliases` 可写行内数组或 Obsidian 多行列表。
- 图片只写 `./文件名`。把 `![[图.png]]` 改成 `![说明](./图.png)`。
- `[[标题或别名]]` 作站内链接。`$…$`、`$$…$$` 直接写。
- `draft: true` 只保存不公开。
- 公开地址 `/blog/post/<slug>`。

## 整理

1. 建 `<slug>/`，正文写入 `index.md`，用到的图片平铺到同层。未被引用的图片不放。
2. 图片文件名改成小写英文加连字符，如 `figure-1.png`，同时改正文引用。
3. Word/pandoc 的 `media/` 子目录图片移到同层，删掉 `{width=...}`。
4. 补 frontmatter。
5. 跑下面的校验。通过后告诉用户文件夹路径和要填写的分类。

除非用户要求，不要直接写进 `Opus/posts/`。正式发布走上传页。

## 校验

```powershell
$dir = '<bundle 路径>'
$slug = Split-Path $dir -Leaf
$items = Get-ChildItem -Force $dir
$img = 'png','jpg','jpeg','gif','webp','avif'
$err = @()
if ($slug -cnotmatch '^[a-z0-9]+(?:-[a-z0-9]+)*$' -or $slug.Length -gt 80) { $err += "slug 不合法: $slug" }
if ($items | Where-Object PSIsContainer) { $err += '存在子文件夹' }
$files = @($items | Where-Object { -not $_.PSIsContainer })
if (@($files | Where-Object Name -ceq 'index.md').Count -ne 1) { $err += '需要恰好一个 index.md' }
foreach ($f in $files) {
  if ($f.Name -ceq 'index.md') { continue }
  if ($f.Extension.TrimStart('.').ToLower() -notin $img) { $err += "不允许的文件: $($f.Name)" }
  if ($f.Name -match '[\s%?#()\[\]]' -or $f.Name.StartsWith('.') -or $f.Name.Length -gt 120) { $err += "文件名不合法: $($f.Name)" }
  if ($f.Length -gt 10MB) { $err += "图片超过 10MB: $($f.Name)" }
}
if ($files.Count -gt 32) { $err += '文件超过 32 个' }
if (($files | Measure-Object Length -Sum).Sum -gt 50MB) { $err += '总大小超过 50MB' }
$md = Get-Content -Raw -Encoding UTF8 (Join-Path $dir 'index.md')
if (-not $md.Trim()) { $err += 'index.md 为空' }
foreach ($m in [regex]::Matches($md, '!\[[^\]]*\]\(\./([^)\s]+)\)')) {
  if (-not (Test-Path -LiteralPath (Join-Path $dir $m.Groups[1].Value))) { $err += "图片引用缺失: $($m.Groups[1].Value)" }
}
$cover = [regex]::Match($md, '(?m)^cover:\s*\.?/?(\S+)')
if ($cover.Success -and -not (Test-Path -LiteralPath (Join-Path $dir $cover.Groups[1].Value))) { $err += "cover 缺失: $($cover.Groups[1].Value)" }
if ($md -match '!\[\[') { $err += '存在 Obsidian 嵌入' }
if ($err) { $err } else { 'OK' }
```

再确认 `Opus/posts/*/<slug>` 不存在。
