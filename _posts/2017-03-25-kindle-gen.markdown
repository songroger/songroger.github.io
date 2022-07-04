---
layout: post
title:  "kindlegen"
date:   2017-03-25 18:15:14 +0000
categories: self words
tail: <a href=https://kindlefere.com/tools#KindleGen>Download</a>
description: 
---
KindleGen 是一个免费的命令行工具，也是亚马逊唯一官方支持的文件转换工具，可通过它把 HTML、XHTML 或 IDPF 2.0 格式（带有 XML.opf 描述文件的 HTML 内容文件）的源文件创建为 Kindle电子图书。高级用户可以使用命令行工具将 EPUB/HTML 转换为 Kindle 电子书。 您可以在 Windows、Mac 和 Linux 平台上使用此界面。此工具可用于自动批量转换。

### KINDLEGEN命令说明

```
*************************************************************
 Amazon kindlegen(MAC OSX) V2.9 build 1028-0897292 
 命令行电子书制作软件 
 Copyright Amazon.com and its Affiliates 2014 
*************************************************************

使用规则：
kindlegen [文件名.opf/.htm/.html/.epub/.zip 或目录] [-c0 或 -c1 或 c2] [-verbose] [-western] [-o <文件名>]

注释：
zip formats are supported for XMDF and FB2 sources
directory formats are supported for XMDF sources

选项: 
-c0：不压缩
-c1：标准 DOC 压缩
-c2：Kindle huffdic 压缩
-o ：指定输出文件名。输出文件将被创建在与输入文件一样的目录中。 不应该包含目录路径。
-verbose： 在电子书转换过程中提供更多信息 
-western：强制创建 Windows-1252 电子书
-releasenotes：显示发行说明
-gif：转换为 GIF 格式的图像（书中没有 JPEG）
-locale  ： 以选定语言显示消息 ( To display messages in selected language ) 
   en: 英语
   de: 德语
   fr: 法语
   it: 意大利语
   es: 西班牙语人
   zh: 中文
   ja: 日本
   pt: 葡萄牙
   ru: Russian
   nl: Dutch
```
除了以上所列出的参数之外，KindleGen 还有一个隐藏参数：-dont_append_source。该参数使得 kindlegen 在生成 mobi 时不再添加源文件到生成的 mobi 文件中，这样可以大大缩减 mobi 的体积，也就不再需要 kindlestrip 来帮助删除 mobi 文件的冗余成分了。具体命令如下所示：
`$ kindlegen -dont_append_source xxx.opf`

### 关于 kindlegen 生成的 mobi 文件
使用 kindlegen 的默认设置生成的 mobi 文件主要包含四部分：

   - 一部分为 MOBI7(azw) 专属文件（html 主文件，内容相关的 opf 文档及目录相关的 ncx 文档）；
   - 一部分为 KF8(azw3) 专属文件（典型的 epub 文件树，包含 css 样式表）；
   - 一部分为 mobi7 和 KF8 格式共用的图片池，包含了所有 html/xhtml 文件链接的图片文件；
   - 最后一部分是转换前的源文件的打包存档（仅供调试之用，推送时不会看到），大小和转换前的 epub 文件相同，这部分对于阅读纯属冗余项，清除对阅读无丝毫影响，kindlestrip 的作用就是将 kindlegen 生成的 mobi 中这部分删除，以求更小的文件体积。

图片池部分有可选的附属部分 —— HD 图片池。当源文件中含大小超过 127KB 的图片时 kindlegen 会自动压缩图片至 127KB 以下（儿童电子书的图片大小为 255KB，这是亚马逊电子书标准所规定的图片体积上限），同时将原图保存在 HD 图片池中（但如果原图超过 2MB 的话还是会压缩至 2MB 以下，2MB 是亚马逊电子书标准中 HD 图片的大小上限）。

云端服务器会识别接收设备，将原始 mobi 文件切分后推送。kindle3 及之前的设备推送 MOBI7(azw) 文件；kindle4 之后的设备推送 KF8(azw3) 文件。MOBI7 格式较简陋，对设备性能要求较低，KF8 格式则更先进，基本支持了 epub 的各个特性，有独立的样式表使得排版更好。这两个文件共同之处在于都使用压缩后的普通图片池以适应电子墨水屏的阅读。而 HD 图片池将在推送至 kindlefire hdx 这样的高清屏设备时，再添加进推送的电子书文件中，以获得更佳的阅读效果。KindleUnpack 中的 HD image 选项正是用 HD 图片（若是有的话）替换压缩后的图片，生成的 epub 中的图片更高清.


### epub常见问题
大部分epub转都会出错，可以用压缩软件打开，修改*.opf文件，补充必要信息再编译即可。
```
   <metadata>
      <meta name="generator" content="Adobe InDesign"/>
      <meta name="cover" content="frontcover.jpg"/>
      <dc:title></dc:title>
      <dc:creator></dc:creator>
      <dc:subject></dc:subject>
      <dc:description></dc:description>
      <dc:publisher></dc:publisher>
      <dc:date></dc:date>
      <dc:source></dc:source>
      <dc:relation></dc:relation>
      <dc:coverage></dc:coverage>
      <dc:rights></dc:rights>
      <dc:language>zh-CN</dc:language>
      <dc:identifier id="bookid"></dc:identifier>
   </metadata>
```