---
title: "[shell] 删除文件夹下部分文件"
author: "wukn"
tags: ["shell"]
catalog: true
date: 2017-11-15T00:00:00+08:00
draft: false
url: /2017/11/15/delete-some-files-in-shell/

---

> 有时候我们会遇到这种情况，需要删除一个文件夹下的部分文件，例如指定后缀的文件、指定前缀的文件等。我们可以使用rm、find、globignore来实现该目标。

<!--more-->

### Linux文件名模式匹配

Linux下shell模式为一个包含以下特殊字符的字符串，称为通配符或者元字符串：

* `*` 匹配0个或多个字符
* `？` 匹配任意一个字符
* `[seq]` 匹配seq中的任意字符
* `[!seq]` 匹配不在seq中的任意字符

### 使用扩展模式匹配操作符删除文件

扩展模式匹配操作符有如下几种，其中`pattern-list`是包含一个或多个文件名的列表，列表中使用`|`分割：

* `*(pattern-list)` 匹配0个或多个出现的指定模式
* `?(pattern-list)` 匹配0个或1个出现的指定模式
* `+(pattern-list)` 匹配1个或多个出现的指定模式
* `@(pattern-list)` 匹配指定模式中的1个
* `!(pattern-list)` 匹配除了指定模式中的任意内容

为了使用这些操作符，需要打开shell的`extglob`选项：

```shell
shopt -s extglob
```

1.删除一个目录下除filename之外的所有文件：

```shell
rm -v !("filename")
```

2.删除一个目录下除filename1和filename2之外的所有文件：

```shell
rm -v !("filename1"|"filename2") 
```

3.删除不是.zip后缀的文件

```shell
rm -v !(*.zip)
```

4.删除不是.zip和.odt后缀的文件

```shell
rm -v !(*.zip|*.odt)
```

使用完了以后记得关闭shell的`extglob`选项：

```shell
shopt -u extglob
```

### 使用find命令删除文件

使用find命令的合适的选项或者使用管道符结合xargs命令也可以实现该功能：

```shell
find /directory/ -type f -not -name 'PATTERN' -delete
find /directory/ -type f -not -name 'PATTERN' -print0 | xargs -0 -I {} rm {}
find /directory/ -type f -not -name 'PATTERN' -print0 | xargs -0 -I {} rm [options] {}
```

5.删除文件夹下除了.gz之外的所有文件

```shell
find . -type f -not -name '*.gz' -delete
```

6.使用管道和xargs删除文件夹下除.gz之外的所有文件

```shell
find . -type f -not -name '*gz' -print0 | xargs -0 -I {} rm -v {}
```

7.删除除了指定后缀的所有文件

```shell
find . -type f -not \(-name '*gz' -or -name '*odt' -or -name '*.jpg' \) -delete
```

### 使用bash的GLOBIGNORE变量删除文件

在bash下，GLOBIGNORE变量存储的是冒号分割的模式列表（文件名），如果一个文件名与路径扩展模式匹配，同时匹配GLOBIGNORE中的一个模式时，它被从匹配列表中删除。

```shell
GLOBIGNORE=*.odt:*.iso:*.txt

rm -v *

unset GLOBIGNORE
```

---

参考资料：

https://www.tecmint.com/delete-all-files-in-directory-except-one-few-file-extensions/

