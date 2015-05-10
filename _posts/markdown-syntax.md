title: "Markdown Cheatsheet"
date: 2015-04-29 20:24:50
category: Techniques
tags: [cheatsheet, lang]
---

![Markdown Logo](logo.jpg)

# 基本Markdown语法

## 标题

<table><thead><tr><th width="45%">Markdown代码</th><th width="5%"></th><th width="50%">格式化输出</th></tr></thead><tbody><tr><td>
<pre>\# 一级标题
\#\# 二级标题
\#\#\#\#\# 五级标题 </pre>
</td><td></td><td><h1>一级标题</h1><h2>二级标题</h2><h5>五级标题</h5></td></tr></tbody></table>

## 列表

<table><thead><tr><th width="45%">Markdown代码</th><th width="5%"></th><th width="50%">格式化输出</th></tr></thead><tbody><tr><td>
<pre>+ 一级表项1
  - 二级表项1
    * 三级表项1
+ 一级表项2
  1. 二级表项1
  1. 二级表项2
+ 一级表项3</pre>
</td><td></td><td><ul><li>一级表项1<ul><li>二级表项1<ul><li>三级表项1</li></ul></li></ul></li><li>一级表项2<ol><li>二级表项1</li><li>二级表项2</li></ol></li><li>一级表项3</li></ul></td></tr></tbody></table>

## 格式化文本

<table><thead><tr><th width="45%">Markdown代码</th><th width="5%"></th><th width="50%">格式化输出</th></tr></thead><tbody><tr><td>
<pre><code>我是 \*斜体\*

我是 \_斜体\_</code></pre>
</td><td></td><td><p>我是 <em>斜体</em></p><p>我是 <em>斜体</em></p></td></tr>
<tr><td>
<pre>我是 \*\*粗体\*\*

我是 \_\_粗体\_\_</pre>
</td><td></td><td><p>我是 <strong>粗体</strong></p><p>我是 <strong>粗体</strong></p></td></tr>
</tbody></table>

## 引用

<table><thead><tr><th width="45%">Markdown代码</th><th width="5%"></th><th width="50%">格式化输出</th></tr></thead><tbody><tr><td>
<pre>> 1st level
> > 2nd level
> back to 1st level</pre>
</td><td></td><td><blockquote><p>1st level</p><blockquote><p>2nd level</p></blockquote><p>back to 1st level</p></blockquote></td></tr></tbody></table>

## 分界线

<table><thead><tr><th width="45%">Markdown代码</th><th width="5%"></th><th width="50%">格式化输出</th></tr></thead><tbody><tr><td>
<pre>\* \* \*
\*\*\*
\*\*\*\*\*
--------------------------------------</pre>
</td><td></td><td><hr><hr><hr><hr></td></tr>
</tbody></table>

## 代码片段

<table><thead><tr><th width="45%">Markdown代码</th><th width="5%"></th><th width="50%">格式化输出</th></tr></thead><tbody><tr><td>
<pre>    This is a code block</pre>
</td><td></td><td><code>This is a code block</code></td></tr>
</tbody></table>

## 链接

<table><thead><tr><th width="45%">Markdown代码</th><th width="5%"></th><th width="50%">格式化输出</th></tr></thead><tbody><tr><td>
<pre>\[看起来是这样\]\(实际地址在这里\)
< 自动链接@example.org ></pre>
</td><td></td><td><a href="实际地址在这里">看起来是这样</a><br><a href="&#x6d;&#x61;&#x69;&#x6c;&#x74;&#x6f;&#x3a;&#x81ea;&#21160;&#38142;&#x63a5;&#x40;&#101;&#x78;&#x61;&#x6d;&#112;&#x6c;&#101;&#46;&#x6f;&#x72;&#x67;">&#x81ea;&#21160;&#38142;&#x63a5;&#x40;&#101;&#x78;&#x61;&#x6d;&#112;&#x6c;&#101;&#46;&#x6f;&#x72;&#x67;</a></td></tr>
</tbody></table>

## 图片

<table><thead><tr><th width="45%">Markdown代码</th><th width="5%"></th><th width="50%">格式化输出</th></tr></thead><tbody><tr><td>
<pre>\!\[描述文字\]\(vi_clippy_assistant.gif\)</pre>
</td><td></td><td><img src="vi_clippy_assistant.gif" alt="描述文字"></td></tr>
</tbody></table>

# GFM（Github Flavored Markdown）扩展语法

## 格式化文本

<table><thead><tr><th width="45%">Markdown代码</th><th width="5%"></th><th width="50%">格式化输出</th></tr></thead><tbody><tr><td>
<pre>ext4\_create\_inode
\_create\_inode
\_create\_</pre>
</td><td></td><td><p>ext4_create_inode</p><p>_create_inode</p><p><em>create</em></td></tr><tr><td>
<pre>\~\~划掉划掉\~\~</pre>
</td><td></td><td><del>划掉划掉</del></td></tr>
</tbody></table>

## 代码片段

<table><thead><tr><th width="45%">Markdown代码</th><th width="5%"></th><th width="50%">格式化输出</th></tr></thead><tbody><tr><td>
<pre>\`\`\`
仍然是个代码片段
但不需要4个空格的缩进
\`\`\`</pre>
</td><td></td><td><figure class="highlight"><table><tr><td class="gutter"><pre><span class="line">1</span><br><span class="line">2</span><br></pre></td><td class="code"><pre><span class="line">仍然是个代码片段</span><br><span class="line">但不需要4个空格的缩进</span><br></pre></td></tr></table></figure></td></tr>
</tbody></table>



## 表格

<table><thead><tr><th width="45%">Markdown代码</th><th width="5%"></th><th width="50%">格式化输出</th></tr></thead><tbody><tr><td>
<pre>左对齐|居中|右对齐
:--|:-:|--:
←|→←|→</pre>
</td><td></td><td><table><thead><tr><th style="text-align:left">左对齐</th><th style="text-align:center">居中</th><th style="text-align:right">右对齐</th></tr></thead><tbody><tr><td style="text-align:left">←</td><td style="text-align:center">→←</td><td style="text-align:right">→</td></tr></tbody></table></td></tr>
</tbody></table>


# References

[Daring Fireball: Markdown Syntax Documentation](http://daringfireball.net/projects/markdown/syntax)
