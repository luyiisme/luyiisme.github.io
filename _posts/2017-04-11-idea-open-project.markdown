---
layout:     post
title:      "命令行快速打开 idea 项目"
date:       2017-04-12
author:     "luyi"
header-img: "img/post-bg-metalworking.jpg"
tags:
    - intellij
---
首先找到并打开(如果你是用的mac os,你可以在 alfred 中快速启动) intellij idea程序，然后在最近打开过的项目历史中找到目标项目，或者
点"open"后，去文件系统目录中去选择。上述过程中打开项目有点麻烦。

这里介绍了一种在命令行里快速打开 idea 项目的方法？比如:

```
//启动idea 应用
idea
```

```
//项目根目录
idea $project.dir
```

```
//打开项目pom.xml
idea $project.pom.file
```
下面是具体实现方式:

- 首先，新建个shell脚本 「idea.sh」;

```
#!/bin/sh

# check for where the latest version of IDEA is installed
IDEA=`ls -1d /Applications/IntelliJ\ * | tail -n1`
wd=`pwd`

# were we given a directory?
if [ -d "$1" ]; then
#  echo "checking for things in the working dir given"
  wd=`ls -1d "$1" | head -n1`
fi

# were we given a file?
if [ -f "$1" ]; then
#  echo "opening '$1'"
  open -a "$IDEA" "$1"
else
    # let's check for stuff in our working directory.
    pushd $wd > /dev/null

    # does our working dir have an .idea directory?
    if [ -d ".idea" ]; then
#      echo "opening via the .idea dir"
      open -a "$IDEA" .

    # is there an IDEA project file?
    elif [ -f *.ipr ]; then
#      echo "opening via the project file"
      open -a "$IDEA" `ls -1d *.ipr | head -n1`

    # Is there a pom.xml?
    elif [ -f pom.xml ]; then
#      echo "importing from pom"
      open -a "$IDEA" "pom.xml"

    # can't do anything smart; just open IDEA
    else
#      echo 'cbf'
      open "$IDEA"
    fi

    popd > /dev/null
fi
```

[内容来源于此](https://gist.github.com/chrisdarroch/7018927)

- 然后 (chmod 该脚本权限后)在 ~/.profile 新增 alias

```
alias idea="sh $dir/idea.sh"
```
---
### 著作权声明

`首次发布于此，转载请保留以上链接`
