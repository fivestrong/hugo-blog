---
title: "React Core Concepts"
date: 2021-02-01T17:34:47+08:00
tags: ["react"]
categories: ["web"]
draft: true
---

## React components 简介

React每一个组件都可以作为用户界面上的一个独立单元，所见即所得，你在组件里面写啥浏览器就会呈现啥效果。为了能够运行不报错，每个组件必须返回至少一个UI元素。

创建React组件的方式有两种，通过JavaScript `class` 和 `function`语法来实现。下面通过 `class` 来定义组件

```react
import React from 'react';

class MainComponent extends React.Component {
  render(){
    return <h1> Hello, World!</h1>
  }
}
```

`class` 组件必须总是包含 render() 函数并且通过 re

