---
layout: post
title: Markdown总结
date: 2019-11-07 
tags: 总结    
---


### 什么是 Markdown  
 
   Markdown 是一种方便记忆、书写的纯文本标记语言，用户可以使用这些标记符号以最小的输入代价生成极富表现力的文档：如您正在阅读的这篇文章。它使用简单的符号标记不同的标题，分割不同的段落，**粗体** 或者 *斜体* 某些文字.

　 很多产品的文档也是用markdown编写的，并且以.MD或者.markdown后缀的文件保存在项目的目录下。    

    Markdown is a way to style text on the web. You control the display of the document; 
    formatting words as bold or italic, adding images, and creating lists are just a few of the things we can do with Markdown.
    Mostly, Markdown is just regular text with a few non-alphabetic characters thrown in, like # or *.                         
 
### [Markdown官方介绍](https://guides.github.com/features/mastering-markdown/)
官方地址：https://guides.github.com/features/mastering-markdown/

### Examples  

#### 1.段落 : 段落之间空一行  

#### 2.换行符 : 一行结束时输入两个空格                 

#### 3.画平行线：三个或者三个以上的 - 或者 * 都可以。  
***
*****

#### 4.字体加粗：  
**bold**

#### 5.字体倾斜：   
*italic*

#### 6.字体倾斜和加粗：  
***bold&italic***

#### 7.链接Links：  
[link to Google!](http://google.com)

#### 8.删除线Strikethrough:  
~~this~~

#### 9.文字大小：  
****
# This is an h1 tag
## This is an h2 tag
### This is an h3 tag 
#### This is an h4 tag 
##### This is an h5 tag 
###### This is an h6 tag 
****

#### 10.无序列表Unordered Lists：无序列表用 - + * 任何一种都可以  
* Item 1
* Item 2
   * Item 2a
   * Item 2b
  
#### 11.有序列表Ordered Lists:数字加点   列表嵌套:上一级和下一级之间敲三个空格即可  
1. Item 1
1. Item 2
1. Item 3
   - Item 3a
   - Item 3b

#### 12.图片Images: 
![GitHub Logo](/images/zhihu.png)  
Format: ![Alt Text](/images/zhihu.png)  
Markdown 还没有办法指定图片的高度与宽度，如果你需要的话，你可以使用普通的 <img> 标签。  
<img src="http://static.runoob.com/images/runoob-logo.png" width="50%">

#### 13.引用块Blockquotes:
> We're living the future so  
> the present is our past.  

#### 14.引用也可以嵌套：  
> 这是引用的内容
>> 这是引用的内容  
>>> 这是引用的内容  


#### 15.内联代码Inline code:  代码之间分别用一个反引号包起来  
内嵌代码`alert('Hello World');`  
I think you should use an `<addr>` element here instead.  
css 的大部分语法同样可以在 markdown 上使用，但不同的渲染器渲染出来的 markdown 内容样式也不一样，下面这些链接里面有 markdown 基本语法

#### 16.Syntax highlighting：
    Here’s an example of how you can use syntax highlighting with GitHub Flavored Markdown
    
    You can also simply indent your code by four spaces  
    
    Python code without syntax highlighting:
        
#### 17.任务清单Task Lists:  
- [x] @mentions, #refs, [links](), **formatting**, and <del>tags</del> supported
- [x] list syntax required (any unordered or ordered list supported)
- [x] this is a complete item
- [ ] this is an incomplete item

#### 18.表格Tables:  
First Header | Second Header  
------------ | -------------  
Content from cell 1 | Content from cell 2  
Content in the first column | Content in the second column

#### 19.SHA引用SHA references:  
16c999e8c71134401a78d4d46435517b2271d6ac  
mojombo@16c999e8c71134401a78d4d46435517b2271d6ac  
mojombo/github-flavored-markdown@16c999e8c71134401a78d4d46435517b2271d6ac  

#### 20.Issue references within a repository:
#1  
mojombo#1  
mojombo/github-flavored-markdown#1  

#### 21.Username @mentions:  
@mention

#### 22.Automatic linking for URLs:  
Any URL (like http://www.github.com/) will be automatically converted into a clickable link.

#### 23.表情符号表情Emoji:  
GitHub supports [emoji](https://help.github.com/en/github/writing-on-github/basic-writing-and-formatting-syntax#using-emoji)!  
To see a list of every image we support, check out the [Emoji Cheat Sheet](https://github.com/ikatyang/emoji-cheat-sheet/blob/master/README.md).

#### 24.流程图：  
```flow
st=>start: 开始
op=>operation: My Operation
cond=>condition: Yes or No?
e=>end
st->op->cond
cond(yes)->e
cond(no)->op
&```