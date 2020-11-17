---
title: Cesium-Pass 渲染对象类型
date: 2020-11-08 12:48:26
tags:
---
&ensp; &ensp; &ensp; 如果理解cesium引擎的渲染模块，最好的入口在哪？我觉得是Context和Pass两个类还有Scene.executeCommands这个方法。Pass中定义了引擎所能渲染的对象类型，cesium不管影像地形倾斜如何调度，最终都是要生成DrawCommand（渲染指令），而这些渲染指令存放在view.frustumCommandsList中（参考《Cesium-Z-Fighting 深度冲突问题》-多视锥体渲染），cesium在Scene.executeCommands方法中会根据pass拿到不同类型的指令去context中执行渲染，不同类型的DrawCommand是有渲染顺序的！

## 1. Pass

``` bash
var Pass = {
  ENVIRONMENT: 0,// 环境
  COMPUTE: 1,// 通用计算
  GLOBE: 2,// 影像 地形
  TERRAIN_CLASSIFICATION: 3,// 地形相关物体
  CESIUM_3D_TILE: 4,// 3D Tiles
  CESIUM_3D_TILE_CLASSIFICATION: 5,// 3D Tiles相关物体
  CESIUM_3D_TILE_CLASSIFICATION_IGNORE_SHOW: 6,
  OPAQUE: 7,// 不透明物体
  TRANSLUCENT: 8,// 半透明物体
  OVERLAY: 9,// 覆盖物
  NUMBER_OF_PASSES: 10,
};
```

&ensp; &ensp; &ensp; 这个类定义了十种渲染类型，ENVIRONMENT背景天空盒大气，COMPUTE通用GPU计算（以后有时间再讨论，涉及到浮点纹理）。GLOBE影像地形，TERRAIN_CLASSIFICATION地形相关物例如贴底线，CESIUM_3D_TILE倾斜摄影，CESIUM_3D_TILE_CLASSIFICATION/CESIUM_3D_TILE_CLASSIFICATION_IGNORE_SHOW倾斜摄影相关物，OPAQUE不透明物体，TRANSLUCENT半透明物体（以后有时间再讨论，涉及到OIT），OVERLAY覆盖物例如标牌

## 2. FrustumCommands

&ensp; &ensp; &ensp; 这个类在多视锥体渲染中涉及到过，挂在view.frustumCommandsList中。在Scene.executeCommands中会遍历frustumCommandsList获取FrustumCommands对象，然后重点来了，根据渲染顺序会通过Pass来在FrustumCommands对象中找到所有DrawCommand。

``` bash
function FrustumCommands(near, far) {
  this.near = defaultValue(near, 0.0);
  this.far = defaultValue(far, 0.0);

  var numPasses = Pass.NUMBER_OF_PASSES;
  var commands = new Array(numPasses);
  var indices = new Array(numPasses);

  for (var i = 0; i < numPasses; ++i) {
    commands[i] = [];
    indices[i] = 0;
  }

  this.commands = commands;
  this.indices = indices;
}
```

&ensp; &ensp; &ensp; 在FrustumCommands类中commands是个数组，这个数组中根据Pass的不同类型存放了不同类型的DrawCommand。而indices则是存放着不同类型DrawCommand的个数！！！最好去打个断点理解一下。

``` bash
us.updatePass(Pass.GLOBE);
var commands = frustumCommands.commands[Pass.GLOBE];
```

## 3.cesium渲染顺序

&ensp; &ensp; &ensp; Scene.executeCommands中是引擎集中处理Command的地方，在这个方法中根据Pass的不同按顺序进行渲染。下面我用伪代码来描述这个过程：

``` bash
function executeCommands(scene, passState) {
----------------------------------------------------------------------
    us.updatePass(Pass.ENVIRONMENT);
    if (!picking) {
    // 天空盒
    var skyBoxCommand = environmentState.skyBoxCommand;
    if (defined(skyBoxCommand)) {
      executeCommand(skyBoxCommand, scene, context, passState);
    }
    // 大气
    if (environmentState.isSkyAtmosphereVisible) {
      executeCommand(
        environmentState.skyAtmosphereCommand,
        scene,
        context,
        passState
      );
    }
    // 太阳
    if (environmentState.isSunVisible) {
      environmentState.sunDrawCommand.execute(context, passState);
    }
    // 月亮
    if (environmentState.isMoonVisible) {
      environmentState.moonCommand.execute(context, passState);
    }
  }
----------------------------------------------------------------------
  // numFrustums多视锥体渲染
  for (var i = 0; i < numFrustums; ++i) {
----------------------------------------------------------------------
    // 绘制地球影像和地形
    us.updatePass(Pass.GLOBE);
    var commands = frustumCommands.commands[Pass.GLOBE];
    var length = frustumCommands.indices[Pass.GLOBE];

    if (globeTranslucent) {
        ......
    } else {
      for (j = 0; j < length; ++j) {
        executeCommand(commands[j], scene, context, passState);
      }
    }
----------------------------------------------------------------------
    // Draw terrain classification 例如 贴地线 贴地面
    if (!environmentState.renderTranslucentDepthForPick) {
      us.updatePass(Pass.TERRAIN_CLASSIFICATION);
      commands = frustumCommands.commands[Pass.TERRAIN_CLASSIFICATION];
      length = frustumCommands.indices[Pass.TERRAIN_CLASSIFICATION];
      if (globeTranslucent) {
        ......
      } else {
        for (j = 0; j < length; ++j) {
          executeCommand(commands[j], scene, context, passState);
        }
      }
    }
----------------------------------------------------------------------
    // 与3D Tiles相关的绘制处理
    if (...) {
      // Draw 3D Tiles
      us.updatePass(Pass.CESIUM_3D_TILE);
      commands = frustumCommands.commands[Pass.CESIUM_3D_TILE];
      length = frustumCommands.indices[Pass.CESIUM_3D_TILE];
      for (j = 0; j < length; ++j) {
        executeCommand(commands[j], scene, context, passState);
      }

      if (length > 0) {
        if (defined(globeDepth) && environmentState.useGlobeDepthFramebuffer) {
          globeDepth.executeUpdateDepth(context, passState, clearGlobeDepth);
        }
        if (!environmentState.renderTranslucentDepthForPick) {
          us.updatePass(Pass.CESIUM_3D_TILE_CLASSIFICATION);
          commands =
            frustumCommands.commands[Pass.CESIUM_3D_TILE_CLASSIFICATION];
          length = frustumCommands.indices[Pass.CESIUM_3D_TILE_CLASSIFICATION];
          for (j = 0; j < length; ++j) {
            executeCommand(commands[j], scene, context, passState);
          }
        }
      }
    }
----------------------------------------------------------------------
    // 不透明物体绘制处理
    us.updatePass(Pass.OPAQUE);
    commands = frustumCommands.commands[Pass.OPAQUE];
    length = frustumCommands.indices[Pass.OPAQUE];
    for (j = 0; j < length; ++j) {
      executeCommand(commands[j], scene, context, passState);
    }
----------------------------------------------------------------------
    // 半透明物体绘制处理
    us.updatePass(Pass.TRANSLUCENT);
    commands = frustumCommands.commands[Pass.TRANSLUCENT];
    commands.length = frustumCommands.indices[Pass.TRANSLUCENT];
    executeTranslucentCommands(
      scene,
      executeCommand,
      passState,
      commands,
      invertClassification
    );
----------------------------------------------------------------------

  }
}
```
&ensp; &ensp; &ensp; 上面描述已经大概说明了cesium的渲染顺序，代码做过精简；其实在整个渲染过程中还涉及到FBO的切换和绑定，尤其是半透明物体OIT的渲染是有些复杂的，还涉及到Scene.resolveFramebuffers方法，threejs中处理半透明是采用Alpha/Blend的方式，这在一些情况下是有问题的。

{% blockquote @superman 1780721345@qq.com %}
文章中有任何错误，请批评勘正
{% endblockquote %}
