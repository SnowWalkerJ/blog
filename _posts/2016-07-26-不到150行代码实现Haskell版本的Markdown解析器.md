---
title: 不到150行代码实现Haskell版本的Markdown解析器
layout: post
tags:
  - Haskell
  - 编程
date: 2016-07-26 20:58:19
---


转到[Github](https://github.com/SnowWalkerJ/Haskell-practice/tree/master/Markdown)查看源代码

# 前言
看到网上有人说Haskell非常适合做解析，它的Parsec库又特别好用。在*Real World Haskell*上学习了用Parsec解析csv和json，感觉真不是一般的灵活和强大。网上也有人用parsec解析LRC歌词的，感觉就有点大材小用了。于是昨天在想，为什么不写个Markdown的解析器呢！

事实证明Parsec确实强大，像加粗、段落、台头这样的上下文无关的标签几乎分分钟就搞定了，50行代码就搞定了。原本以为处理列表和表格会特别困难，要用到State之类的，确实也花了点时间，但后来发现其实也非常简单,配合上css，就能让格式变得非常好看。后续工作我就没做下去。

当然，代码高亮和数学公式我都没做，要做这两个就不只是Markdown语法那么简单了，而且代码高亮可以借助JS库完成。

<!-- more -->

# 代码解析
Parsec库自带了很多解析组建，我们要做的就是把这些组建拼装起来，组成我们自己的解析器。

### 先来最简单的*斜体*解析器：

```haskell
italicParser::CharParser () String
italicParser = do
    char '*'
    content <- many (noneOf "*\n")
    char '*'
    return $ "<i>" ++ content ++ "</i>"
```
首先一个`char '*'`说明要求匹配一个星号作为开始，然后`content <- many (noneOf "*\n")`表示要匹配一串不包含星号和回车的字符串，并将匹配结果传给content， 再一个`char '*'`表示匹配一个星号作为结束，最后在`content`内容前后加上`<i></i>`标签返回。

### **粗体**解析器也是同理：
```haskell
boldParser::CharParser () String
boldParser = do
    string "**"
    content <- many (noneOf "*\n")
    string "**"
    return $ "<b>" ++ content ++ "</b>"
```
因为这次匹配的是双星号，所以用string而不是char。

### 正文解析器：
```haskell
contentParser = do
    thisParsed <- try boldParser 
	    <|> try italicParser 
	    <|> do {c<-noneOf "\n"; return [c]}
    nextParsed <- contentParser <|> return ""
    return $ thisParsed ++ nextParsed
```
首先尝试匹配粗体，如果匹配失败则尝试匹配斜体，再失败的话就匹配单个非换行字符；然后再调用自身匹配接下去的内容，直到匹配失败（遇到换行符），把所有匹配到的内容连接起来。

### 标题解析器
```haskell
headParser = do
    many (oneOf " \t")
    sharps <- many1 (char '#')
    try (char ' ') <|> return ' '
    let n = length sharps
    content <- contentParser
    return $ "<h" ++ show n ++"><b>" ++ content ++ "</b></h" ++ show n ++ ">\n"
```
首先把行首的空格和制表符去掉，然后匹配至少一个井号，再尝试匹配1个或0个空格，然后使用正文解析器解析标题的内容，以井号的个数来判断标题的级别，最后以`<h1></h1>`标签包装好后输出。

### 段落解析器
```haskell
paragraphParser = do
    content <- contentParser
    return $ "<p>" ++ content ++ "</p>\n"
```
这个很简单，就是用正文解析器匹配一整行的内容，然后用`<p></p>`包装起来就好了。

### 分隔符解析器
```haskell
hrParser = atLeast 5 (char '-') >> return "<hr />\n"
```
如果匹配到至少五个`-`则返回`<hr />`标签
其中`atLeast`解析器是这样定义的：
```haskell
atLeast n ps = do
    s <- many ps
    if length s >= n then return s
    else fail ""
```
匹配尽可能多个ps，如果匹配到的数量不小于n则返回匹配到的数组，否则匹配失败。

这样就把最简单的几个解析器都完成了，接下来要做的列表解析器和表格解析器会复杂一些。

## 列表解析器
在html里列表有两种标签组成：`<ul>`和`<li>`，前者表示一个列表，后者表示列表中的每一个单项。
先看parser计数器：
```haskell
countParser parser = do
    s <- many parser
    return $ length s
```
尽可能匹配一个解析器，然后返回匹配到的次数。这个计数器下面会用到。

### ul解析器：
```haskell
ulParser upLevel = do
    n <- countParser (char '\t')
    if n < upLevel then fail "" else char '-'
    many $ oneOf " \t"
    thisItem <- contentParser
    char '\n'
    items <- many (try (liParser n) <|> try(ulParser n)) <|> return [""]
    let indent = replicate n '\t'
        itemHTML = indent ++ "\t<li>" ++ thisItem ++ "</li>\n"
    return $ indent ++ "<ul>\n" ++ itemHTML ++ concat items ++ indent ++ "</ul>\n"
```
记录匹配到的开头的tab数量，作为当前列表的级别。如果当前缩进级别小于上层的级别说明上一级别的列表结束了，让自己匹配失败，否则的话匹配一个`-`来保证当前依旧是一个列表。
第5行匹配若干个空格和制表符，把这些都去掉。然后用正文解析器解析当前项目的内容，传给thisItem， 再匹配一个回车作为结束。
再不停使用`liParser`和`ulParser`匹配剩余的项目和子列表，直到匹配失败，再将它们连接在一起，添加HTML标签，加一些缩进让代码好看一点。

### li解析器
写得好累啊。。。`liParser`和`ulParser`很像，更简单一些，就当个习题留下不表了，读者自己动动脑筋吧。

## 表格解析器

```haskell
data Align = AlignLeft | AlignCenter | AlignRight
instance Show Align where
    show AlignLeft = "left"
    show AlignCenter = "center"
    show AlignRight = "right"
tableParser::CharParser () String
tableParser = do
    titles  <- titleParser
    char '\n'
    aligns  <- alignParser
    char '\n'
    rows    <- sepEndBy rowParser (char '\n')
    let tr row = "<tr>\n" ++ concatMap td alignedRow ++ "</tr>\n"
            where alignedRow = zip aligns row
        td (align, col) = "\t<td class=\"" ++ show align ++ "\">" ++ col ++ "</td>\n"
        th (align, col) = "\t<th class=\"" ++ show align ++ "\">" ++ col ++ "</th>\n"
    return $ "<table border=\"1\" cellspacing=\"0\" cellpadding=\"4\">\n<tr>\n" 
	    ++ concatMap th (zip aligns titles) 
	    ++ "</tr>\n" 
	    ++ concatMap tr rows 
	    ++ "</table>"
```
定义了Align结构表示表格内容的对齐方式。
表格解析器调用了`titleParse`,`alignParser`和`rowParser`分别解析表格的标题、对齐方式和正文内容，然后用HTML代码连接起来。

```haskell
titleParser::CharParser () [String]
titleParser = char '|' >> endBy (many (noneOf "|\n")) (char '|')
```

```haskell
alignParser = char '|' >> endBy (do
    left <- countParser (char ':')
    many1 (char '-')
    right <- countParser (char ':')
    return $ if left > 0 
        then if right > 0 
            then AlignCenter
            else AlignLeft
        else AlignRight
    ) (char '|')
```

```haskell
rowParser = char '|' >> endBy (many (noneOf "|\n")) (char '|')
```

## 其他
```haskell
html content = "<!DOCTYPE html>\n<html>\n" ++ header ++ body content ++ "</html>"
header = "<head>\n" ++ 
    "<style type=\"text/css\">\n" ++
    ".center {text-align: center;}\n" ++
    ".left {text-align: left;}\n" ++
    ".right {text-align: right;}\n" ++
    "table {border-collapse:collapse;}</style>\n</head>\n"
body content = "<body>" ++ content ++ "</body>"
```
这里定义了一些HTML标签和CSS样式，主要是控制表格的对齐。
```haskell
parser = do 
    parsed <- flip sepEndBy (char '\n') $ 
        try hrParser
        <|> try tableParser
        <|> try (ulParser 0)
        <|> try headParser 
        <|> try paragraphParser
        <|> return ""
    return $ concat parsed
```
把上面定义的解析器依次连接起来，形成一个总的Markdown解析器。
```haskell
parseMarkdown markdown = 
    let parsed = parse parser "ParseError" markdown
    in case parsed of
        Left err -> "ParseError"
        Right result -> html result
```
调用Markdown解析器来解析目标Markdown代码。
整个过程非常简单直观。
