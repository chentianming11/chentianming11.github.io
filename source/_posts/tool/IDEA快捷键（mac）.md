---
title: IDEA快捷键（mac）
tags: idea
categories: tool
abbrlink: 81062828
date: 2021-05-14 19:48:22
---

自用的`mac`版本IDEA快捷键。

<!--more-->

## 无处不在的跳转

`command + 数字键` ：打开数字标识的侧边栏或者是底栏。

`alt + command +  [ 或 ]`：切换到上一个项目窗口。

`shift + command + A` : 查找命令

`command + E` : 最近打开的文件

`shift + command + E` : 最近编辑的文件

`shift + command + delete` ：跳转到上一个编辑的位置

`alt + command + <-  /  ->`  : 跳转到  上一个  / 下一个  浏览的位置

`F2` : 跳转到错误的位置。

`F11` : 添加到书签

`command + F11 + 数字` : 添加到书签x

`ctrl + 数字` ：跳转到对应书签

`shift + F11` : 显示所有书签

`alt + shift + F`：添加到收藏。可以是类或者是方法。

`command + G`：跳转指定行数

## 精准搜索

`command + N` ：搜索类

`shift + command + N` ：搜索文件

`ctrl + shift + ccommand + N` ：搜索符号（匹配方法名或者字段名）

`ctrl + shift + f`：可以搜索项目下、模块下或者指定目录下的字符串

`command+ shift+ +/-`：折叠代码

## live template

`psvm` : 生成main方法

`iter` : 迭代

`psf` ：public static final

`psfi` : public static final Integer $var1$ = $var2$;

`psfs` : public static final String $var1$ = "$var2$";

`ps` : private String $var1$ = "$var2$";

`psc` ：带注释版

`pi` : private Integer $var1$ = $var2$;

`pic` ：带注释版

`st` : String

`tna` ：throw new AppException("$var1$");

## postfix

`.fori`

`.iter`

`.nn`

`.if`

`.sout`

`.return`

`.not`

`.null`

`alt + enter`：神器，不同的位置有不同的功能。


## 重构

`shift + F6`：重命名

`command+ F6`：方法重构

## 抽取

`alt + command + v`：补全代码；抽取为方法变量；

`alt + command + C`：抽取为静态成员变量；

`alt + command + F`：抽取为成员变量

`alt + command+ P`：抽取成方法参数

`alt + command+ M`：抽取成一个方法；

## 编辑

`command + Z`：撤销

`shift +command  + Z`：还原

`alt +command + L`：格式化代码

`alt +command + O`：优化导包

`command + F`：查找文本

`command+ R`：替换文本

`alt + /`：代码提示

`command+ X`：剪切行

`command+ Y`：删除行

`command+ D`：复制行

`command+ /`：单行注释

`shift +command + /` ：多行注释

`shift +command + up/down`：选中的代码上下移动

`command+ F4`：关闭当前标签页

`shift +command + t`：为一个类创建测试用例

`shift +command + U`：大小写转换

`SHIFT + ALT + U`: 驼峰，下划线，中杆格式转化

`command + alt + T`：使用语句块(try catch, if esle...)包围代码

`ctrl + N` ：智能插入

`ctrl + command + G` ： 列操作

`alt + ->` : 光标移动到单词结尾

`shift + alt + ->` : 选中到单词结尾

`command + ->` : 光标移动到行尾

`shift + command + ->` : 选中到行尾

`command + .` : 折叠代码

## 调试

`command+ F8`：添加断点

`shift + F9`：debug运行

`shift + F10`：run运行

`ctrl + alt + R`：jrebel的debug运行

`F7`：进入方法

`F8`：单步

`F9`：Resume（跳到下一个断点）

`shift +command + F8`：查看所有断点。在断点位置，测试设置断点的条件

`alt + F8`：表达式求值

`alt + F9`：运行到光标处

在debug时，选中变量按`F2 : `为该变量设值。

9. 结构图
   
`command + F12`：查看结构（如类的方法和成员变量）

`alt + shift + command + U`：显示结构图（maven的依赖图 或者是  类图）

`ctrl + H`：显示类的层级关系。

`ctrl +alt + H`：显示方法的调用关系

## 其他

`F5`：复制文件到当前目录

`F6`：移动文件到指定目录

`command + c` ： 复制文件名

`shift + comand + c` ： 复制文件完整路径名

`alt + shift + comand + c` ：复制类的引用路径

`command + ,`  :  打开设置

`command + ;` : 打开项目设置
