---
layout:     post
title:      深入理解AsyncTask
subtitle:   AsyncTask的实现原理
date:       2018-07-06
author:     BY
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Android 知识总结
    - 多线程
    - 源码解析
---
# 前言

>目前我们在项目中的异步操作大多是使用RxJava来实现。但是仍然有必要理解AsyncTask的实现原理及优缺点。

# 常见操作方法介绍

#### 方法
execute
executeOnExecutor
cancel

#### 回调

onPreExecute：
doInBackground：
onProgressUpdate：
onPostExecute：
onCancelled：
#### 操作思想

等赋值之后在去做事情。


