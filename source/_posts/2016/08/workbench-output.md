---
title: workbench只导出数据（含insert语句）
date: 2016-08-27 20:28:28
categories: mysql
tags: [mysql,workbench,ubuntu,数据,数据库]
---

# 1. 说明：
**出发点：**
　　由于特殊原因，我们**只想**导出数据库中的数据（`insert into`语句格式的），但是在网上找到的资源很少（关于linux），因此特撰此文。
<!--more-->
# 2. 环境
1. *mysql*
2. *mysql workbench*
3. *ubuntu (linux)*

# 3. 开始
1. 打开**workbench**,连接上数据库后进入主界面；
2. 在左侧导航栏中，点击*MANAGEMENT*中的`Data Export`；
3. 默认`Object Selection`页面中，在`Select Database Objects to Export`栏里勾选你要导出的数据库；
4. **重头戏**
 1. 点击右上方的`Advanced  Options...`按钮，进入高级设置界面；
 2. 在`Inserts`栏中，勾选`extended-insert`（本表所有数据只用一条`insert into`语句，如果你想一张表中的每条数据都对应着一个`insert into`语句，你可以取消勾选本选项，勾选上面的`compelete-insert`）；
 3. 在`SQL`栏中，勾选`comments`和`quote-names`选项；
 4. 在`Tables`栏中，**务必**勾选上`no-create-info`选项；
 5. 点击**右上角**的`< Return`按钮，回到初始界面；
5. 在`Options`栏中，点击`Export to Self-Contained File`，在后面输入导出路径即可；
6. 点击右下角`Start Export`按钮，导出成功；

# 4. 结语
　　本文在实际工程中有应用到，希望能够帮助到大家。ps:同时，这也算是正式进军*Markdown*的写作行列的*milestone*。图片暂时没时间加，更新的时候再弄上去吧。^_^
