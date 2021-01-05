---
title: Cesium-地球瓦片遮挡计算
date: 2021-01-04 15:07:45
tags:
---
&ensp; &ensp; &ensp; 关于在地球上判断一个Tile（或者其他几何体）是否可见的计算是十分必要的，尤其是在GIS引擎当中，如何能精准的计算出一个Tile是否可见决定着与这个Tile相关的资源（纹理切片/地形数据）是否下载以及这个Tile是否需要渲染。由于地球的影响，视锥体剔除这种过滤方式不能够满足Tile是否可见的判断，因为即使在视锥体之内，地球背面的Tile是无法断定是否可见的。因此Horizon Culling（习惯叫做“地平线剔除”）应运而生。

## 1. Horizon Culling

&ensp; &ensp; &ensp; 如下图所示，白实线为相机视锥体，白虚线为viewer（相机）与地球切线（地平线），红点位于视锥体之外，绿点和蓝点位于视锥体之内，但是绿点都是可见的（没有地球遮挡），蓝点虽然在视锥体之内，但是却被地球遮挡了（即不可见也就是不可渲染），那么如何判定蓝点是否可见呢？

![剔除](shizhuiti.jpg)

&ensp; &ensp; &ensp; 如下图所示，首先将地球抽象为一个单位球（unit-sphere即半径为1，至于如何抽象下面会详细介绍），由viewer发出的两条射线与球向切（地平线），实际上与单位球相切的线会有无数条，它们组成一个圆锥（自行脑补）。图中灰色阴影部分就是从viewer方向看被单位球遮挡住的部分。可以这样说，位于圆锥的底面之前的是可见的，位于之后的是不可见的，问题就变成了“一个点在一个面的前面还是后面”。

![构建数学计算模型](shuxuemoxing.jpg)

&ensp; &ensp; &ensp; 如下图所示，T为处于视锥内的任一点，要想判断T是在圆锥底面之前还是之后很简单，就看VT在VC方向上的投影距离VQ，如果VQ > VP则该点被地球遮蔽，如果VQ < VP则该点未被遮蔽；具体推到计算过程其实是很简单的，因为这个球是单位球即HC=1，V是相机位置，C为地球原点，把VP求出来应该算是初一下学期期末考试的题吧？求出VT的投影VQ应该是算是胎教题吧？
&ensp; &ensp; &ensp; 能够判定视锥体内任意一个点在圆锥底面的前后就万无一失了吗？当然不是！看一看第一张图最下边那个绿点，向VC投影之后在要大于VP，但是它可没有被遮挡！所以还需要一个判断，判断“一个点是否在圆锥中，这个圆锥是个底部无限延长的”。

![面测试](mianjisuan.jpg)

&ensp; &ensp; &ensp; 如下图所示，判断一个点是否在无限锥之内，只需要比夹角大小就可以了。至于计算太简单了就不用说了！

![无限锥测试](zhuijisuan.jpg)

&ensp; &ensp; &ensp; 一个十分重要的问题：上文总是说单位球，为什么要构造单位球呢？定义一个三维球的标准方程：(x - a)2 + (y - b)2 + (z - c)2 = r2，单位球方程为：x2 + y2 + z2 = 1，定义一个单位椭球的方程为：x2/a2 + y2/b2 + z2/c2 = 1，由公式可以看出(x/a)2 + (y/b)2 + (z/c)2 = 1, 单位椭球其实从某种意义上说是单位球在xyz轴上的缩放，将一个坐标的xyz分别除以椭球的三个轴半径即可将椭球坐标转换到单位球坐标系中（Cesium称为椭球缩放坐标系 ellipsoid-scaled space），代码如下：

``` bash
// Ellipsoid radii - WGS84 shown here
var rX = 6378137.0;
var rY = 6378137.0;
var rZ = 6356752.3142451793;

// Vector CV
var cvX = cameraPosition.x / rX;
var cvY = cameraPosition.y / rY;
var cvZ = cameraPosition.z / rZ;

// VH
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

// 通过判断是否在无限锥内外和圆锥底面前后来决定是否遮蔽
var isOccluded = vtDotVc > vhMagnitudeSquared && 
                vtDotVc * vtDotVc / vtMagnitudeSquared > vhMagnitudeSquared;
```

## 2. Computing the horizon occlusion point

&ensp; &ensp; &ensp; 上文我们仅仅是判断一个点在viewer方向上是否被地球遮蔽，那么问题来了，如果是一个地形Tile呢？总不能遍历这个Tile所有的顶点来判断其是否被遮蔽吧？Cesium中的解决办法是将Tile计算出一个遮蔽点，通过这个遮蔽点来参与第一节的计算，就能够判断这个Tile是否被遮挡。记得某位前辈曾说过，图形学搞到底全都是数学问题，这在Cesium中体现的淋漓尽致。

![遮蔽点计算](goujian.jpg)

&ensp; &ensp; &ensp; 如上图所示，蓝色球为单位球（unit-sphere），棕色几何体为地形Tile，OC是从单位球心到Tile包围体（可能是包围球也可能是旋转包围盒，不过这都不重要，我们只要其center点）中心点C发出的一条射线，V是Tile上的一个顶点，H是V与单位球的切线，其实V与单位球有无数个切线，但是只有两条切线会与OC相交（H-V-P和虚线），通过遍历Tile所有顶点计算顶点与单位球切线和OC的交点，找出最大的OP距离，这个最大的OP距离的P交点，即为所要得到的 horizon occlusion point。

![遮蔽点计算](zhebidian.jpg)

&ensp; &ensp; &ensp; 如上图所示，关于遮蔽点的具体计算无非就是求切线求交点等问题，很简单的立体几何计算，此处就不啰里啰唆了。为什么说最大的OP即在OC方向上最远的交点P就是遮蔽点呢？为什么通过判断这个点就可以决定这个Tile是否被地球遮挡呢？我们可以想象一下，从Viewer方向看，一个Tile是否被地球遮蔽，取决于一个很要的东西：切线（地平线），主动权在Viewer这里，现在换个思路，如果我们从Tile的每个顶点看相机呢？最早看到相机的那个顶点与单位球的切线HV与OC的交点就是P！相机最早看到P就相当于看到Tile的V点了！

``` bash
// EllipsoidalOccluder.js
/**
 * 计算遮蔽点（注意需要转换到椭球缩放空间）
 * @param ellipsoid WGS84椭球对象
 * @param directionToPoint Tile中心点
 * @param positions Tile的四个点（此处注意上文说到遍历Tile的顶点是不对的，
 *   首先此处尚且没有Tile的网格数据，其次遍历顶点性能太差）
 */
function computeHorizonCullingPointFromPositions(
  ellipsoid,
  directionToPoint,
  positions,
  result
) {

  if (!defined(result)) {
    result = new Cartesian3();
  }

  var scaledSpaceDirectionToPoint = computeScaledSpaceDirectionToPoint(
    ellipsoid,
    directionToPoint
  );
  var resultMagnitude = 0.0;

  for (var i = 0, len = positions.length; i < len; ++i) {
    var position = positions[i];
    var candidateMagnitude = computeMagnitude(
      ellipsoid,
      position,
      scaledSpaceDirectionToPoint
    );
    if (candidateMagnitude < 0.0) {
      // all points should face the same direction, but this one doesnt, so return undefined
      return undefined;
    }
    resultMagnitude = Math.max(resultMagnitude, candidateMagnitude);// 找出最大OP
  }

  return magnitudeToPoint(scaledSpaceDirectionToPoint, resultMagnitude, result);
}

```

&ensp; &ensp; &ensp; 对于上述代码，只提醒一点：主要将坐标转换到椭球缩放空间，算完之后在转换回去！！！

## 3. 总结

&ensp; &ensp; &ensp; 在文章开头就说过，以上计算解决了Tile在视锥体之内的地球遮蔽问题，能够判定在视锥体之内，从相机方向看地球背面瓦片是否被遮挡，从而决定Tile是否加载资源是否渲染等，这个过滤很重要！但是在Tile真正组装成DrawCommand之后，这个渲染指令会不会渲染还要经过判断！这跟相机视锥体/Tile包围球/occluder有关，具体计算原理和过程以后再讨论，代码如下：

``` bash
Scene.prototype.isVisible = function (command, cullingVolume, occluder) {
  return (
    defined(command) &&
    (!defined(command.boundingVolume) ||
      !command.cull ||
      (cullingVolume.computeVisibility(command.boundingVolume) !==
        Intersect.OUTSIDE &&
        (!defined(occluder) ||
          !command.occlude ||
          !command.boundingVolume.isOccluded(occluder))))
  );
};
```

&ensp; &ensp; &ensp; 本篇博客参考Cesium两篇文章，特此注明！

{% blockquote @superman 1780721345@qq.com %}
文章中有任何错误，请批评勘正
{% endblockquote %}
