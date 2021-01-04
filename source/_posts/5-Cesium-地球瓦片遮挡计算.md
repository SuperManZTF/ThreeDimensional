---
title: Cesium-地球瓦片遮挡计算
date: 2021-01-04 15:07:45
tags:
---
&ensp; &ensp; &ensp; 关于在地球上判断一个Tile（或者其他几何体）是否可见的计算是十分必要的，尤其是在GIS引擎当中，如何能精准的计算出一个Tile是否可见决定着与这个Tile相关的资源（纹理切片/地形数据）是否下载以及这个Tile是否需要渲染。由于地球的影响，视锥体剔除这种过滤方式不能够满足Tile是否可见的判断，因为即使在视锥体之内，地球背面的Tile是无法断定是否可见的。因此Horizon Culling（习惯叫做“地平线剔除”）应运而生。

## 1. Horizon Culling

&ensp; &ensp; &ensp; 如下图所示，白实线为相机视锥体，白虚线为viewer（相机）与地球切线（地平线），红点位于视锥体之外，绿点和蓝点位于视锥体之内，但是绿点都是可见的（没有地球遮挡），蓝点虽然在视锥体之内，但是却被地球遮挡了（即不可见也就是不可渲染），那么如何判定蓝点是否可见呢？

![剔除](shizhuiti.jpg)

&ensp; &ensp; &ensp; 如下图所示，首先将地球抽象为一个单位球（unit-sphere即半径为1），由viewer发出的两条射线与球向切（地平线），实际上与单位球相切的线会有无数条，它们组成一个圆锥（自行脑补）。图中灰色阴影部分就是从viewer方向看被单位球遮挡住的部分。可以这样说，位于圆锥的底面之前的是可见的，位于之后的是不可见的，问题就变成了“一个点在一个面的前面还是后面”。

![构建数学计算模型](shuxuemoxing.jpg)

&ensp; &ensp; &ensp; 如下图所示，T为处于视锥内的任一点，要想判断T是在圆锥底面之前还是之后很简单，就看VT在VC方向上的投影距离VQ，如果VQ > VP则该点被地球遮蔽，如果VQ < VP则该点未被遮蔽；具体推到计算过程其实是很简单的，因为这个球是单位球即HC=1，V是相机位置，C为地球原点，把VP求出来应该算是初一下学期期末考试的题吧？求出VT的投影VQ应该是算是胎教题吧？
&ensp; &ensp; &ensp; 能够判定视锥体内任意一个点在圆锥底面的前后就万无一失了吗？当然不是！看一看第一张图最下边那个绿点，向VC投影之后在要大于VP，但是它可没有被遮挡！所以还需要一个判断，判断“一个点是否在圆锥中，这个圆锥是个底部无限延长的”。

![面测试](mianjisuan.jpg)

&ensp; &ensp; &ensp; 如下图所示，判断一个点是否在无限锥之内，只需要比夹角大小就可以了。至于计算太简单了就不用说了！

![无限锥测试](zhuijisuan.jpg)

&ensp; &ensp; &ensp; 一个十分重要的问题：上文总是说单位球，为什么要构造单位球呢？定义一个三维球的标准方程：(x - a)2 + (y - b)2 + (z - c)2 = r2，单位球方程为：x2 + y2 + z2 = 1，定义一个单位椭球的方程为：x2/a2 + y2/b2 + z2/c2 = 1，由公式可以看出(x/a)2 + (y/b)2 + (z/c)2 = 1,单位椭球其实从某种意义上说是单位球在xyz轴上的缩放。

``` bash
// Ellipsoid radii - WGS84 shown here
var rX = 6378137.0;
var rY = 6378137.0;
var rZ = 6356752.3142451793;

// Vector CV
var cvX = cameraPosition.x / rX;
var cvY = cameraPosition.y / rY;
var cvZ = cameraPosition.z / rZ;

var vhMagnitudeSquared = cvX * cvX + cvY * cvY + cvZ * cvZ - 1.0;
```

``` bash
// Target position, transformed to scaled space
var tX = position.x / rX;
var tY = position.y / rY;
var tZ = position.z / rZ;

// Vector VT
var vtX = tX - cvX;
var vtY = tY - cvY;
var vtZ = tZ - cvZ;
var vtMagnitudeSquared = vtX * vtX + vtY * vtY + vtZ * vtZ;

// VT dot VC is the inverse of VT dot CV
var vtDotVc = -(vtX * cvX + vtY * cvY + vtZ * cvZ);

var isOccluded = vtDotVc > vhMagnitudeSquared && 
                vtDotVc * vtDotVc / vtMagnitudeSquared > vhMagnitudeSquared;
```

## 2. Computing the horizon occlusion point

## 3. occluder遮蔽

## 4. 总结 
