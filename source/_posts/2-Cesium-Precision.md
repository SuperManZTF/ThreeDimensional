---
title: Cesium-Precision 精度问题
date: 2020-11-08 12:47:06
tags:
---
&ensp; &ensp; &ensp; 在cesium引擎中处理地球所用的数据都非常大，JS能支持Float64数据精度，但是webgl仅支持Float32精度，所以当由CPU向GPU提交顶点属性数据会产生“精度丢失”，反映在渲染图像上是”闪烁/抖动“等。解决数据精度丢失问题的常用方法有两种：RTC（Relative To Center）和RTE（Relative To Eye），cesium中使用的是RTE这种方式。

## 1. RTC

&ensp; &ensp; &ensp; Relative To Center简称RTC，相信大家对webgl的几个重要坐标系都很熟悉（本地坐标系/世界坐标系/相机坐标系/投影坐标系），假如需要构建一个立方体，我们描述顶点的时候不使用世界坐标系下的点来构建（原点在世界坐标系中心），而是选择立方体的中心点作为原点，立方体顶点都是相对中心点给值（值会变小），将立方体放到中心点所在的世界坐标系中，这样的效果会和直接给定立方体世界坐标系下的顶点数据相同！但是顶点数据变小了。
&ensp; &ensp; &ensp; 有人可能会问，即使立方体的顶点数据是相对于中心点给定，值确实会变小，但是要想将立方体放到世界坐标系下某位置，需要左乘M矩阵（模型矩阵），这个模型矩阵中的位移部分也是个大值，精度还是会丢失？实际上确实是这样，这个立方体构建的M矩阵位移部分确实值会很大，但是我们可以不需要直接往着色器中传递M矩阵，而是传递MV矩阵！这个矩阵会把M中位移部分的大值抵消掉，在着色器中直接左乘MV矩阵。 
&ensp; &ensp; &ensp; 特别需要注意：虽然MV矩阵抵消了大的值，但是当相机离立方体很远的时候还是由精度丢失的情况，但是由于渲染对象离得比较远，细节关注度大大下降，所以也没啥问题。一个渲染对象，坐标值非常大，把这些值相对于一个中心点来给定，然后左成MV矩阵不直接左乘M矩阵，在相机离得比较近的时候精度丢失问题可以解决。

## 2. RTE

&ensp; &ensp; &ensp; Relative To Eye简称RTE，这种方式和RTC有相似之处，上面顶点数据是相对于中心点给定，而RTE的顶点数据是相对于相机的（Eye）。下面是cesium源码中处理RTE过程，AutomaticUniforms是cesium中自动计算uniform值的类，它会在每帧每个command执行的时候根据着色器需要传入的uniform值的不同而去计算相应uniform，下文中需要的uniform值都在其中自行断点查看。

``` bash
function EncodedCartesian3() {
  this.high = Cartesian3.clone(Cartesian3.ZERO);
  this.low = Cartesian3.clone(Cartesian3.ZERO);
}

/**
 * @example
 * var value = 1234567.1234567;
 * var splitValue = Cesium.EncodedCartesian3.encode(value);
 */
EncodedCartesian3.encode = function (value, result) {
  //>>includeStart('debug', pragmas.debug);
  Check.typeOf.number("value", value);
  //>>includeEnd('debug');

  if (!defined(result)) {
    result = {
      high: 0.0,
      low: 0.0,
    };
  }

  var doubleHigh;
  if (value >= 0.0) {
    doubleHigh = Math.floor(value / 65536.0) * 65536.0;
    result.high = doubleHigh;
    result.low = value - doubleHigh;
  } else {
    doubleHigh = Math.floor(-value / 65536.0) * 65536.0;
    result.high = -doubleHigh;
    result.low = value + doubleHigh;
  }

  return result;
};
```

&ensp; &ensp; &ensp; 上面这段代码可谓是实现GPU双精度计算的精髓！将一个数值拆分成high和low两部分，即将一个Float64的值拆分成两个Float32的值；上文说过，js支持Float64，webgl仅支持Float32！这样拆分之后往webgl中传递数据精度可以保证了。

``` bash

uniform mat4 czm_modelViewRelativeToEye;
attribute vec3 positionHigh;
attribute vec3 positionLow;

vec4 czm_translateRelativeToEye(vec3 high, vec3 low)
{
  vec3 highDifference = high - czm_encodedCameraPositionMCHigh;
  vec3 lowDifference = low - czm_encodedCameraPositionMCLow;

  return vec4(highDifference + lowDifference, 1.0);
}

void main()
{
  vec4 p = czm_translateRelativeToEye(positionHigh, positionLow);
  gl_Position = czm_projection * (czm_modelViewRelativeToEye * p);
}

```

&ensp; &ensp; &ensp; czm_modelViewRelativeToEye：相对于相机（Eye）的MV矩阵，左乘可以变换到相机空间
&ensp; &ensp; &ensp; positionHigh/positionLow：一个Float64拆分成两个Float32的值
&ensp; &ensp; &ensp; czm_encodedCameraPositionMCHigh/czm_encodedCameraPositionMCLow：相机位置（一个Float64拆分成两个Float32的值），一定要注意这个相机位置的坐标系奥！MC MC MC
&ensp; &ensp; &ensp; czm_projection：相机投影矩阵
&ensp; &ensp; &ensp; 本来在这一段我想详细理一下过程，但是感觉说多了也没用，懂得已经懂了！上面的计算和数据传递，全程避免了大数值！

``` bash
function cleanModelViewRelativeToEye(uniformState) {
  if (uniformState._modelViewRelativeToEyeDirty) {
    uniformState._modelViewRelativeToEyeDirty = false;

    var mv = uniformState.modelView;
    var mvRte = uniformState._modelViewRelativeToEye;
    mvRte[0] = mv[0];
    mvRte[1] = mv[1];
    mvRte[2] = mv[2];
    mvRte[3] = mv[3];
    mvRte[4] = mv[4];
    mvRte[5] = mv[5];
    mvRte[6] = mv[6];
    mvRte[7] = mv[7];
    mvRte[8] = mv[8];
    mvRte[9] = mv[9];
    mvRte[10] = mv[10];
    mvRte[11] = mv[11];
    mvRte[12] = 0.0;
    mvRte[13] = 0.0;
    mvRte[14] = 0.0;
    mvRte[15] = mv[15];
  }
}
```

&ensp; &ensp; &ensp; 最后有必要说一下czm_modelViewRelativeToEye这个矩阵是如何构建的，很简单就是MV矩阵去掉位移部分，为什么？因为着色器czm_translateRelativeToEye方法中已经减去相机位置了！！！
&ensp; &ensp; &ensp; 这里再次提醒czm_encodedCameraPositionMCHigh和czm_encodedCameraPositionMCLow一定要注意坐标系！！！

{% blockquote @superman 1780721345@qq.com %}
文章中有任何错误，请批评勘正
{% endblockquote %}
