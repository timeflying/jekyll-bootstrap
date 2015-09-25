---
layout: post
title: "Hello World from Jason"
description: ""
category: 
tags: []
---
{% include JB/setup %}

#A quick reference of __Markdown__#

- - - 

_Markdown_ is powerful, and I only write down part of it __base on personal preference__.

The reference of this doc is [here](https://github.com/othree/markdown-syntax-zhtw/blob/master/syntax.md#link/).

*   [Title](#title)
*   [List](#list)
*   [Divider](#divider)
*   [Link](#link)
*   [Code](#code)
*   [Inline code](#inline-code)
*   [Emphasize](#emphasize)
*   [Blockquote](#blockquote)
*   [Image](#image)

- - - 

<h2 id="title">Title</h2>

You just put `#` at the beginning and the end of the line, like this: `# This is an H1 #` or `## This will be an H2 ##`

#### This is a H4 ####

- - -

<h2 id="list">List</h2>

If you want an unordered list, you need `- Red`.

-   Red
-   Green
-   Blue

On the other hand, for an ordered list, you need `1. Bird`.

1.  Bird
2.  McHale
3.  Parish

-   A list item with a blockquote:

    > This is a blockquote
    > inside a list item.

-   A list item with a code block:

        <code goes here>

- - -

<h2 id="divider">Divider</h2>

It's simple, like this `- - -`.

- - -
<br>
<br>

- - -

<h2 id="link">Link</h2>

`This is a link to [Andromoney](http://web.andromoney.com/) inline link.`

This is a link to [Andromoney](http://web.andromoney.com/) inline link.

- - -

<h2 id="code">Code</h2>

A `tab` at the beginning of the line will make it as a code.

This is a normal paragraph:

    This is a code block.

Here is an example of AppleScript:

    tell application "Foo"
        beep
    end tell

- - -

<h2 id="inline-code">Inline code</h2>

Use `` ` `` to quote the inline code.

`` `printf()` `` will show `printf()`.

- - -

<h2 id="emphasize">Emphasize</h2>

`_single asterisks_` will show:

_single asterisks_		

`__double underscores__` will show:

__double underscores__

- - -

<h2 id="blockquote">Blockquote</h2>

A `>` is put at the beginning of the line, like this: `> This is a blockquote with`.

> This is a blockquote with two paragraphs. Lorem ipsum dolor sit amet,
consectetuer adipiscing elit. Aliquam hendrerit mi posuere lectus.
Vestibulum enim wisi, viverra nec, fringilla in, laoreet vitae, risus.


> #### This is a H4 header. ####
> 
> 1.   This is the first list item.
> 2.   This is the second list item.
> 
> Here's some example code:
> 
>     return shell_exec("echo $input | $markdown_script");

- - -

<h2 id="image">Image</h2>

![Markdown](http://markdown.tw/images/208x128.png)