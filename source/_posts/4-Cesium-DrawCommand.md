---
title: Cesium-DrawCommand
date: 2020-11-08 12:52:34
tags:
---
&ensp; &ensp; &ensp; 在Cesium当中不管影像/地形/倾斜摄影如何调度，还是在地球之上任何绘制几何体，最终都会组装成一个DrawCommand渲染指令；然后将DrawCommand送入Context中执行渲染任务。

## 1. DrawCommand中有什么？

``` bash
function DrawCommand(options) {
  this._boundingVolume = options.boundingVolume;// 包围球
  this._orientedBoundingBox = options.orientedBoundingBox;// 旋转包围盒
  this._cull = defaultValue(options.cull, true);// 是否进行视锥剔除，这涉及到View.js里面的指令过滤
  this._occlude = defaultValue(options.occlude, true);// 是否进行地球遮蔽判断，这涉及到View.js里面的指令过滤
  this._modelMatrix = options.modelMatrix;// 模型矩阵，对，就是MVP的M
  this._primitiveType = defaultValue(// WebGL渲染几何体的类型 点 线 面...
    options.primitiveType,
    PrimitiveType.TRIANGLES
  );
  this._vertexArray = options.vertexArray;// 顶点属性数据 VAO
  this._count = options.count;// 顶点数量
  this._offset = defaultValue(options.offset, 0);// 顶点偏移
  this._instanceCount = defaultValue(options.instanceCount, 0);// 实例化渲染数量
  this._shaderProgram = options.shaderProgram;// 着色程序
  this._uniformMap = options.uniformMap;// 纹理贴图
  this._renderState = options.renderState;// 渲染状态：深度/模板/颜色缓冲操作设置（例如 禁止模板测试 禁止颜色写入等）
  this._framebuffer = options.framebuffer;// 当前DrawCommand渲染所用的FBO
  this._pass = options.pass;// 当前DrawCommand所属的Pass，Pass用来控制先后渲染顺序
  this._executeInClosestFrustum = defaultValue(
    options.executeInClosestFrustum,
    false
  );
  this._owner = options.owner;// 所对应的逻辑对象,例如某个地块的DrawCommand属于QuadtreeTile实例
  this._debugShowBoundingVolume = defaultValue(// 是否调试包围体，会打开这个DrawCommand包围体
    options.debugShowBoundingVolume,
    false
  );
  this._debugOverlappingFrustums = 0;
  this._castShadows = defaultValue(options.castShadows, false);// 产生阴影
  this._receiveShadows = defaultValue(options.receiveShadows, false);// 接受阴影
  this._pickId = options.pickId;// 拾取ID，这里涉及到Cesium中的GPUPicker拾取机制
  this._pickOnly = defaultValue(options.pickOnly, false);// 是否只用于拾取，这里涉及到Cesium中的GPUPicker拾取机制

  this.dirty = true;
  this.lastDirtyTime = 0;

  /**
   * @private
   */
  this.derivedCommands = {};// 衍生指令，这里涉及到HDR/对数深度/阴影的处理，在原来指令的基础上衍生其他功能，例如增加shader代码块
}
```

&ensp; &ensp; &ensp; 由上面的代码以及注释可以看出，DrawCommand中包含了渲染一个对象所需的各种信息，顶点属性/FBO/Depth||Color||Stencil/纹理/Shader等。将这个类的实例执行excute方法即可使用Context对其进行渲染，Context封装了WebGL上下文，是直接面向GL的。

``` bash
/**
 * Executes the draw command.
 *
 * @param {Context} context The renderer context in which to draw.
 * @param {PassState} [passState] The state for the current render pass.
 */
DrawCommand.prototype.execute = function (context, passState) {
  context.draw(this, passState);
};
```

&ensp; &ensp; &ensp; 上面代码中尤其值得注意的是passState这个参数，这个对象很重要！！！在DrawCommand中有一个_framebuffer变量，当这个变量不为Undefined时，说明这个渲染指令将绘制到挂载的这个FBO中，否则将绘制到passState中的FBO上。

{% blockquote @superman 1780721345@qq.com %}
文章中有任何错误，请批评勘正
{% endblockquote %}
