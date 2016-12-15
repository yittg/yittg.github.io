---
layout: post
title: 为什么老是 ^M
tags: [git]
---

在 Windows 上 `git diff` 的时候偶尔会发现在一些变更行尾的地方出现了奇怪的 `^M`，看着着实有些难受。事实上，之前大概找过解决办法，但对于他们究竟是什么，解决办法究竟做了些什么还是有些模糊。然后今天又遇到了 :( ，决定好好的把它搞搞清楚，实在受不了啦。

### ^M 究竟是什么

其实很简单，在 Windows，Mac，和 Linux 上使用着不同的行尾符，有的是 `CRLF`，有的是`LF`，Windows 使用前者，而 Mac 和 Linux 通常是使用后者。 而 `CR` 对应的 ASCII 码 13（0xD, carriage return)，展示出来就是 `^M`（可以通过 `Ctrl + V, Ctrl + M` 输入），`LF` 则是 10（0xA, line feed）即换行符。

也就是说 `^M` 就是行尾符的一种或者一部分。

### 什么时候会出现（using git）

其实在 diff 的时候只要存在以 `CRLF` 的行变更都应该有 `^M`。之所以有这个疑问，是因为对以 `CRLF` 作为行尾符（add / commit 的时候保留了 `CR`）的文件进行编辑的时候，看到的 `git diff` 的结果中只有 `add lines` 才有明显的 `^M` 字符，看起来就好像是这次编辑引入了这个字符的感觉，真是悲伤。

```diff
diff --git a/file_with_crlf_line_ending b/file_with_crlf_line_ending
index 89bde95..d2084b4 100644
--- a/file_with_crlf_line_ending
+++ b/file_with_crlf_line_ending
@@ -1 +1 @@
-line one
+line one add some space     ^M
```

然而事实是什么呢，事实上只是 git 在显示 diff 的时候只对 `add lines` 末尾的空白符高亮显示（知道真相的我眼泪掉下来）。有一个很方便的验证方式就是反过来 diff，就能看得很清楚了。

```diff
$ git diff -R
diff --git b/file_with_crlf_line_ending a/file_with_crlf_line_ending
index d2084b4..89bde95 100644
--- b/file_with_crlf_line_ending
+++ a/file_with_crlf_line_ending
@@ -1 +1 @@
-line one add some space
+line one^M
```

### 那么如何干掉它呢

_解决方案其实已经很多了，随手一搜就能找到。_

有一个眼不见心不烦的方法，它的意思是说把 `CR` 看作行尾符的一部分。

```
$ git config --global core.whitespace cr-at-eol
```

根本的办法是在 commit 的时候就不要带上 `CR` 字符，这个 git 提供了自动转换的方式。

```
$ git config --global core.autocrlf true
```

> Windows 安装 git 的时候有个选项: __Checkout Windows-style, commit Unix-style line endings__ 也是这个效果。

但对于已经 commit 文件的行尾符中的 `CR` 还得手动去掉（以下方式来自 [stackoverflow.com](http://stackoverflow.com/questions/1889559/git-diff-to-ignore-m)）。

```
# Remove everything from the index
$ git rm --cached -r .

# Re-add all the deleted files to the index
$ git diff --cached --name-only -z | xargs -0 git add

# Commit
$ git commit -m "Fix CRLF"
```

### 呼！

好像终于搞清楚了。

