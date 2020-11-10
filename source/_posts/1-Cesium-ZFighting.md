---
title: Cesium-Z-Fighting 深度冲突问题
---
&ensp; &ensp; &ensp; 在Cesium引擎中深度冲突（Z-Fighting）是绕不开的。之所以会产生深度冲突，是因为两个表面过于接近，深度缓冲区有限的精度已经不能区分哪个在前哪个在后。解决深度冲突问题普遍的方法有“对数深度”和“多边形偏移”，在Cesium当中解决该问题有两种方式“多视锥体”和“多视锥体+对数深度”。
&ensp; &ensp; &ensp; GIS-地球渲染的是行星级别的比例尺，用米度量的分辨率会出现深度冲突和抖动的现象。这里作者使用的是Cesium-1.71版本。关于深度冲突本质上是数值的精度问题，Cesium引擎中处理精度问题还有两种方式RTC和RTE，请结合《Cesium-Precision 精度问题》一文来理解。

## 1. 正常情况下Shader中深度如何计算？

``` bash
// vertex shader
void main(){
    vec3 transformed = vec3( position );
    vec4 mvPosition = vec4( transformed, 1.0 );
    mvPosition = modelViewMatrix * mvPosition;
    gl_Position = projectionMatrix * mvPosition;
}
```

``` bash
// fragment shader
void mian(){
    gl_FragColor = vec4( 1.0,0.0,0.0,1.0 ); 
}
```

&ensp; &ensp; &ensp; 在顶点着色器中做了两个矩阵变换（本质上是三个MV乘起来了）：modelViewMatrix和projectionMatrix，将顶点从局部坐标系转到世界坐标系转到相机坐标系转到投影坐标系，这里的projectionMatrix矩阵是根据相机的视锥体构建的（构建过程参考ThreeJS-Camera），经过矩阵计算gl_Position是被转换到投影剪裁空间（具体参考冯乐乐《Unity Shader入门精要》第四章），此处只说结论：经过矩阵变换最终写入深度缓冲区的深度值是“片元距离相机视锥体近截面的距离”（这个结论如果理解不了就先暂时接受并大胆认可它）。

``` bash
// program
gl.getExtension('EXT_frag_depth');
// fragment shader
void main() {
  gl_FragColor = vec4(1.0, 0.0, 1.0, 1.0); 
  gl_FragDepthEXT = 0.5; 
}
```

&ensp; &ensp; &ensp; 在WebGL shader中如果想在片元着色器中修改片元的深度需要用到WebGL扩展EXT_frag_depth，使用gl_FragDepthEXT可以修改片元的深度值，后边使用对数深度修改片元深度的时候会用到这个扩展，在Cesium中这个扩展及其重要，例如贴地线/贴地面等都会涉及到深度值的修改。

## 2. 多边形偏移

``` bash
// program
gl.enable(gl.POLYGON_OFFSET_FILL);// 启用多边形偏移
gl.polygonOffset(1.0, 1.0);
```

&ensp; &ensp; &ensp; 开启多边形偏移之后，设置多边形偏移参数：gl.polygonOffset(factor, units), 指定加到每个顶点z值上的偏移量，偏移量按m*factor+r*units来计算，m表示顶点所在表面相对于相机视线的角度，r表示硬件能够区分两个z值之差的最小值。这种处理深度冲突的方式在深度值精度够用的情况下是可以的，但是解决不了精度不够用的情况。

## 3. 多视锥体渲染

&ensp; &ensp; &ensp; 多视锥体渲染的关键是根据frameState.commandList来计算视锥体的远近截面和视锥体数量，并将frameState.commandList分类存放到view.frustumCommandsList。在Cesium-view中View.prototype.createPotentiallyVisibleSet是关键方法。
&ensp; &ensp; &ensp; view类中的frustumCommandsList在scene.executeCommands中会遍历并执行其中的DrawCommand，这个数组中根据视锥体数量保存着FrustumCommands（其中根据Pass保存渲染不同类型的对象的DrawCommand）。相信阅读这篇博客的同仁是熟悉Cesium渲染流程的，此处不在赘述，如果不理解请在scene.executeCommands函数中自行断点。由于该方法涉及的内容很多，这里我先贴出代码注释，接下来会把涉及到的每个点进行讲解，最后再整体总结一遍流程。

``` bash
View.prototype.createPotentiallyVisibleSet = function (scene) {
  var frameState = scene.frameState;
  var camera = frameState.camera;
  var direction = camera.directionWC;// 世界坐标下的相机方向
  var position = camera.positionWC;// 世界坐标下的相机位置

  var computeList = scene._computeCommandList;// 计算指令
  var overlayList = scene._overlayCommandList;// 覆盖指令
  var commandList = frameState.commandList;// 与视锥体有关的指令

  // 调试“多视锥体”
  if (scene.debugShowFrustums) {
    ......
  }

  // 视锥体指令集合 索引置零
  var frustumCommandsList = this.frustumCommandsList;
  var numberOfFrustums = frustumCommandsList.length;
  var numberOfPasses = Pass.NUMBER_OF_PASSES;
  for (var n = 0; n < numberOfFrustums; ++n) {
    for (var p = 0; p < numberOfPasses; ++p) {
      frustumCommandsList[n].indices[p] = 0;
    }
  }

  // 通用计算指令清除
  computeList.length = 0;
  // 覆盖指令清除
  overlayList.length = 0;

  var commandExtents = this._commandExtents;
  var commandExtentCapacity = commandExtents.length;
  var commandExtentCount = 0;

  // JS 数值最大值 加减 1
  var near = +Number.MAX_VALUE;
  var far = -Number.MAX_VALUE;

  var shadowsEnabled = frameState.shadowState.shadowsEnabled;
  var shadowNear = +Number.MAX_VALUE;
  var shadowFar = -Number.MAX_VALUE;
  var shadowClosestObjectSize = Number.MAX_VALUE;

  // 判断地球是否遮挡物体
  var occluder = frameState.mode === SceneMode.SCENE3D ? frameState.occluder : undefined;
  // 相机视锥体
  var cullingVolume = frameState.cullingVolume;

  // get user culling volume minus the far plane.除掉视锥体远平面
  var planes = scratchCullingVolume.planes;
  for (var k = 0; k < 5; ++k) {
    planes[k] = cullingVolume.planes[k];
  }
  cullingVolume = scratchCullingVolume;

  var length = commandList.length;
  for (var i = 0; i < length; ++i) {
    var command = commandList[i];
    var pass = command.pass;

    if (pass === Pass.COMPUTE) {// 计算指令
      computeList.push(command);
    } else if (pass === Pass.OVERLAY) {// 覆盖指令
      overlayList.push(command);
    } else {
      var commandNear;
      var commandFar;

      var boundingVolume = command.boundingVolume;// 渲染指令包围盒
      if (defined(boundingVolume)) {
        if (!scene.isVisible(command, cullingVolume, occluder)) {// 判断显示隐藏
          continue;
        }

        // 包围球的圆心和相机位置的射线 向 相机方向 投影 dot算出 一段距离 -> 加减 半径
        var nearFarInterval = boundingVolume.computePlaneDistances(
          position,
          direction,
          scratchNearFarInterval
        );
        commandNear = nearFarInterval.start;
        commandFar = nearFarInterval.stop;

        // console.log('near: ' + commandNear + 'far: ' + commandFar);

        near = Math.min(near, commandNear);// 根据JS最大小值限制
        far = Math.max(far, commandFar);

        // Compute a tight near and far plane for commands that receive shadows. This helps compute
        // good splits for cascaded shadow maps. Ignore commands that exceed the maximum distance.
        // When moving the camera low LOD globe tiles begin to load, whose bounding volumes
        // throw off the near/far fitting for the shadow map. Only update for globe tiles that the
        // camera is not inside.
        if (
          shadowsEnabled &&
          command.receiveShadows &&
          commandNear < ShadowMap.MAXIMUM_DISTANCE &&
          !(pass === Pass.GLOBE && commandNear < -100.0 && commandFar > 100.0)
        ) {
          // Get the smallest bounding volume the camera is near. This is used to place more shadow detail near the object.
          var size = commandFar - commandNear;
          if (pass !== Pass.GLOBE && commandNear < 100.0) {
            shadowClosestObjectSize = Math.min(shadowClosestObjectSize, size);
          }
          shadowNear = Math.min(shadowNear, commandNear);
          shadowFar = Math.max(shadowFar, commandFar);
        }
      } else if (command instanceof ClearCommand) {
        // Clear commands dont need a bounding volume - just add the clear to all frustums.
        commandNear = camera.frustum.near;
        commandFar = camera.frustum.far;
      } else {
        // If command has no bounding volume we need to use the camera s
        // worst-case near and far planes to avoid clipping something important.
        commandNear = camera.frustum.near;
        commandFar = camera.frustum.far;
        near = Math.min(near, commandNear);
        far = Math.max(far, commandFar);
      }

      var extent = commandExtents[commandExtentCount];
      if (!defined(extent)) {
        extent = commandExtents[commandExtentCount] = new CommandExtent();
      }
      extent.command = command;
      extent.near = commandNear;
      extent.far = commandFar;
      commandExtentCount++;
    }
  }

  if (shadowsEnabled) {
    shadowNear = Math.min(
      Math.max(shadowNear, camera.frustum.near),
      camera.frustum.far
    );
    shadowFar = Math.max(Math.min(shadowFar, camera.frustum.far), shadowNear);
  }

  // Use the computed near and far for shadows
  if (shadowsEnabled) {
    frameState.shadowState.nearPlane = shadowNear;
    frameState.shadowState.farPlane = shadowFar;
    frameState.shadowState.closestObjectSize = shadowClosestObjectSize;
  }

  updateFrustums(this, scene, near, far);

  var c;
  var ce;

  for (c = 0; c < commandExtentCount; c++) {
    ce = commandExtents[c];
    insertIntoBin(this, scene, ce.command, ce.near, ce.far);
  }

  // Dereference old commands
  if (commandExtentCount < commandExtentCapacity) {
    for (c = commandExtentCount; c < commandExtentCapacity; c++) {
      ce = commandExtents[c];
      if (!defined(ce.command)) {
        // If the command is undefined, its assumed that all
        // subsequent commmands were set to undefined as well,
        // so no need to loop over them all
        break;
      }
      ce.command = undefined;
    }
  }

  var numFrustums = frustumCommandsList.length;
  var frustumSplits = frameState.frustumSplits;
  frustumSplits.length = numFrustums + 1;
  for (var j = 0; j < numFrustums; ++j) {
    frustumSplits[j] = frustumCommandsList[j].near;
    if (j === numFrustums - 1) {
      frustumSplits[j + 1] = frustumCommandsList[j].far;
    }
  }

```

&ensp; &ensp; &ensp; 重要的事情说三遍：这个方法很重要X3！
&ensp; &ensp; &ensp; 首先提前说明两个事情：frameState中的occluder和cullingVolume是干什么的？
&ensp; &ensp; &ensp; Occluder: 由位置和半径来定义一个球Sphere再结合相机的位置，来计算一个点或者一个包围球等是否被球Sphere遮挡（具体代码详见Cesium-Occluder类），结果有三种。这里在frameState上的occluder是使用地球和相机共同定义的，用于判断某个物体（使用其包围球计算）与地球的遮挡情况。

``` bash
var Visibility = {
  /**
   * Represents that no part of an object is visible.
   */
  NONE: -1,

  /**
   * Represents that part, but not all, of an object is visible
   */
  PARTIAL: 0,

  /**
   * Represents that an object is visible in its entirety.
   */
  FULL: 1,
};
```

&ensp; &ensp; &ensp; CullingVolume：由若干个面定义的体用于计算剔除。每个面是一个Cartesian4对象，xyz代表平面法线，w代表平面与原点的距离，这与ThreeJS当中Plane的定义是一样的。这里在frameState上的cullingVolume是由相机计算而来的（是视锥体，但用法有故事）。

``` bash
/**
 * The culling volume defined by planes. 
 *
 * @alias CullingVolume
 * @constructor
 *
 * @param {Cartesian4[]} [planes] An array of clipping planes.
 */
function CullingVolume(planes) {
  /**
   * Each plane is represented by a Cartesian4 object, where the x, y, and z components
   * define the unit vector normal to the plane, and the w component is the distance of the
   * plane from the origin.
   * @type {Cartesian4[]}
   * @default []
   */
  this.planes = defaultValue(planes, []);
}

```

&ensp; &ensp; &ensp; 在DrawCommand中会有boundingVolume（包围球），scene.isVisible(command, cullingVolume, occluder)会计算判断一个DrawCommand是否可以在此帧渲染，这个判断至关重要！

``` bash
Scene.prototype.isVisible = function (command, cullingVolume, occluder) {
  return (
    defined(command) &&
    (!defined(command.boundingVolume) || !command.cull ||
      (cullingVolume.computeVisibility(command.boundingVolume) !==
        Intersect.OUTSIDE &&
        (!defined(occluder) ||
          !command.occlude ||
          !command.boundingVolume.isOccluded(occluder))))
  );
};
```

&ensp; &ensp; &ensp; 在上面代码中有两个计算函数：cullingVolume.computeVisibility和command.boundingVolume.isOccluded，在上面已经讲过cullingVolume和occluder的定义以及用途，此处不再赘述。此处得出一个重要结论：(!scene.isVisible(command, cullingVolume, occluder))为false，DrawCommand就是可以渲染的! 这里需要说明一下，cullingVolume.computeVisibility的计算结果有三种情况，如下代码：

``` bash
var Intersect = {
  /**
   * Represents that an object is not contained within the frustum.
   *
   * @type {Number}
   * @constant
   */
  OUTSIDE: -1,

  /**
   * Represents that an object intersects one of the frustum s planes.
   *
   * @type {Number}
   * @constant
   */
  INTERSECTING: 0,

  /**
   * Represents that an object is fully within the frustum.
   *
   * @type {Number}
   * @constant
   */
  INSIDE: 1,
};
```

&ensp; &ensp; &ensp; 如果一个DrawCommand是可以渲染的，接下来它就将参与计算相机的远近截面。这个地方的远近截面是所有可渲染的DrawCommand计算出的commandNear和commandFar的累计，累计规则：选出near的最小值和far的最大值。每个DrawCommand是如何计算出commandNear和commandFar呢？

``` bash
BoundingSphere.computePlaneDistances = function (
  sphere,
  position,
  direction,
  result
) {
  if (!defined(result)) {
    result = new Interval();
  }

  var toCenter = Cartesian3.subtract(
    sphere.center,
    position,
    scratchCartesian3
  );
  var mag = Cartesian3.dot(direction, toCenter);

  result.start = mag - sphere.radius;
  result.stop = mag + sphere.radius;
  return result;
};
```

&ensp; &ensp; &ensp; 上面函数中有四个参数：sphere（DrawCommand包围球）position（相机世界坐标位置）direction（相机世界坐标朝向）result（返回结果），注意这里调用这个方法计算传入的参数含义只是在当前计算环境下传参意义，如果此函数用作他用另当别论。渲染指令的包围球由center和radius定义，即包围球位置和半径；通过center和position构建一条向量N，N向direction做投影P（点积计算），投影P在direction方向上的投影“覆盖”距离加上radius就是commandFar，减去radius就是commandNear，如图示。
&ensp; &ensp; &ensp; 
