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
&ensp; &ensp; &ensp; 已经计算出了每个可渲染的DrawCommand以及累计得到了near和far，接下来就要确定视锥体的数量updateFrustums；由下面代码可以看出，根据near和far计算视锥体数量就靠一个公式：numFrustums = Math.ceil(Math.log(far / near) / Math.log(farToNearRatio))，这个公式将其输入到Graph中看一下函数图像，视锥体数量分为三个：1到1000米、1000到100万、100万到100亿之间，如图示。有了视锥体数量之后就可以创建FrustumCommands并放到view.frustumCommandsList中，上文已经说过了，view.frustumCommandsList在scene.executeCommands中遍历执行。

``` bash
function updateFrustums(view, scene, near, far) {
  var frameState = scene.frameState;
  var camera = frameState.camera;
  var farToNearRatio = frameState.useLogDepth
    ? scene.logarithmicDepthFarToNearRatio
    : scene.farToNearRatio;
  var is2D = scene.mode === SceneMode.SCENE2D;
  var nearToFarDistance2D = scene.nearToFarDistance2D;

  // The computed near plane must be between the user defined near and far planes.
  // The computed far plane must between the user defined far and computed near.
  // This will handle the case where the computed near plane is further than the user defined far plane.
  near = Math.min(Math.max(near, camera.frustum.near), camera.frustum.far);
  far = Math.max(Math.min(far, camera.frustum.far), near);

  var numFrustums;
  if (is2D) {
    ......
  } else {
    // The multifrustum for 3D/CV is non-uniformly distributed.
    numFrustums = Math.ceil(Math.log(far / near) / Math.log(farToNearRatio));
  }

  var frustumCommandsList = view.frustumCommandsList;
  frustumCommandsList.length = numFrustums;
  for (var m = 0; m < numFrustums; ++m) {
    var curNear;
    var curFar;

    if (is2D) {
      ......
    } else {
      curNear = Math.max(near, Math.pow(farToNearRatio, m) * near);
      curFar = Math.min(far, farToNearRatio * curNear);
    }
    var frustumCommands = frustumCommandsList[m];
    if (!defined(frustumCommands)) {
      frustumCommands = frustumCommandsList[m] = new FrustumCommands(
        curNear,
        curFar
      );
    } else {
      frustumCommands.near = curNear;
      frustumCommands.far = curFar;
    }
  }
}
```

&ensp; &ensp; &ensp; 在View.prototype.createPotentiallyVisibleSet方法中有必要说一下_commandExtents这个数组，其实很简单就是在DrawCommand真正加入到view.frustumCommandsList中之前临时存放的数组，数组中的每个CommandExtent存放了DrawCommand和DrawCommand的near/far（准确说是commandNear/commandFar），根据视锥体数量来将_commandExtents数组中的DrawCommand分别添加到view.frustumCommandsList的不同视锥体指令数组之中。

``` bash
function CommandExtent() {
  this.command = undefined;
  this.near = undefined;
  this.far = undefined;
}
```

&ensp; &ensp; &ensp; 至此，View.prototype.createPotentiallyVisibleSet方法中计算视锥体数量和相机远近截面的重要点都已经介绍了，接下来整体的讲一遍流程（一定要去打个断点再结合本文来理解）：

1. 获取frameState.commandList（在这一帧渲染之前所有的DrawCommand都将存在这个数组）；scene._computeCommandList和scene._overlayCommandList中分别用来存放通用GPU计算的Command（以后再讨论这个东西）和覆盖物的Command。
2. 获取cullingVolume和occluder，对cullingVolume进行操作，总起来一句话：视锥体由六个面构成，去掉远截面。
3. 遍历frameState.commandList数组，根据command.pass来区分是什么类型的渲染指令，pass === Pass.COMPUTE就将command添加到scene._computeCommandList中，pass === Pass.OVERLAY就将command添加到scene._overlayCommandList，这两个类型比较特殊。其他的command都将进行if/else判断：有无boundingVolume（包围球）—>是否是ClearCommand清除指令—>以及其他：
-------------
有无boundingVolume（包围球）：执行这个条件的代码是最复杂的；首先根据boundingVolume/cullingVolume/occluder对进行command进行选择，scene.isVisible在上文已经讲过了，被留下来的都是可渲染的command，根据包围球和相机位置方向计算commandNear/commandFar，参与near/far累计，然后被添加进commandExtents数组中。
是否是ClearCommand清除指令：清除指令则直接将command和相机的near/far添加进commandExtents数组中，不参与near/far累计。
以及其他：执行这个条件的代码是参与near/far累计操作的，它的累计是使用的相机near/far，然后添加进commandExtents数组中。
-------------
4.累计完near/far，并将所有符合条件的command添加到commandExtents之后，就将进行视锥体个数的计算updateFrustums，创建对应的FrustumCommands并存在view.frustumCommandsList之中。
5.遍历commandExtents数组将其中的command分别存储到不同的视锥体中FrustumCommands，注意：如果一个command被两个视锥体同时占有，则需要分别加入到两个视锥体指令之中，此处也是多视锥体渲染解决深度问题的性能问题所在，因为跨视锥体的command需要渲染两次。
6.最后将commandExtents数组进行清除操作，将所有视锥体的远近截面值存储到frameState.frustumSplits数组中（主要用于调试用）；提示：看这部分源码会注意到，代码中有关于shadow的near/far相关计算，因为阴影的生成也是与相机相关的，near/far是必须的，这里就不讨论了。
&ensp; &ensp; &ensp; 总结：文首说过“多视椎体渲染”是为了解决深度冲突的问题；Cesium通过所有的的DrawCommand累计出相机最终的near和far，并通过一个公式确定在这一帧需要多少个视椎体进行渲染，然后将command分别存储在不同视椎体中，上面我们提到过一个结论：经过矩阵变换最终写入深度缓冲区的深度值是“片元距离相机视锥体近截面的距离”；也就是说相机的near越接近command，command的片元深度值就越小；所以累计计算near和far以及视椎体数量都是为了使command离相机的near面更近，深度值精度更高！如果有多个视椎体的话，cesium会遍历所有视椎体（最多三个），从后往前渲染，最远处的视椎体渲染完之后，清除深度，再使用下一个视椎体渲染，渲染完清除深度以此类推。
## 4. 对数深度
&ensp; &ensp; &ensp; 在第一小节中提到了WebGL扩展EXT_frag_depth，Cesium中的对数深度需要这个扩展（其他基于WebGL的引擎的对数深度实现都需要），因为在片元着色器中需要使用gl_FragDepthEXT来修改即将写入ZBuffer中的深度值。
```bash
// 顶点着色器

varying float v_depthFromNearPlusOne;
uniform vec2 czm_currentFrustum;// vec2(near,far)
vec4 czm_updatePositionDepth(vec4 coords) {
   coords.z = clamp(coords.z / coords.w, -1.0, 1.0) * coords.w;
   return coords;
}
void czm_vertexLogDepth()
{
   v_depthFromNearPlusOne = (gl_Position.w - czm_currentFrustum.x) + 1.0;
   gl_Position = czm_updatePositionDepth(gl_Position);
}
```
```bash
// 片元着色器

varying float v_depthFromNearPlusOne;
uniform float czm_farDepthFromNearPlusOne;// far -near + 1.0
uniform float czm_oneOverLog2FarDepthFromNearPlusOne;

void czm_writeLogDepth(float depth)
{
   if (depth <= 0.9999999 || depth > czm_farDepthFromNearPlusOne) {
      discard;
   }

   gl_FragDepthEXT = log2(depth) * czm_oneOverLog2FarDepthFromNearPlusOne;
}

void czm_writeLogDepth() {
   czm_writeLogDepth(v_depthFromNearPlusOne);
}
```
&ensp; &ensp; &ensp;上面代码是顶点着色器和片元着色器与对数深度相关的代码，在Cesium-Shader中这块代码还有其他处理，为了方便理解我做过精简。这里我要做一个说明：在上一节中我们计算过多视椎体，在使用对数深度开启时（scene.logarithmicDepthBuffer = true）,多视椎体的计算还是需要的，只不过参与计算的系数farToNearRatio不同(numFrustums = Math.ceil(Math.log(far / near) / Math.log(farToNearRatio)))。通过logarithmicDepthFarToNearRatio参与计算之后视椎体数量会减少，上面有张图可以说明。
```bash
/**
   * The far-to-near ratio of the multi-frustum when using a normal depth buffer.
   * <p>
   * This value is used to create the near and far values for each frustum of the multi-frustum. It is only used
   * when {@link Scene#logarithmicDepthBuffer} is <code>false</code>. When <code>logarithmicDepthBuffer</code> is
   * <code>true</code>, use {@link Scene#logarithmicDepthFarToNearRatio}.
   * </p>
   *
   * @type {Number}
   * @default 1000.0
   */
this.farToNearRatio = 1000.0;

/**
   * The far-to-near ratio of the multi-frustum when using a logarithmic depth buffer.
   * <p>
   * This value is used to create the near and far values for each frustum of the multi-frustum. It is only used
   * when {@link Scene#logarithmicDepthBuffer} is <code>true</code>. When <code>logarithmicDepthBuffer</code> is
   * <code>false</code>, use {@link Scene#farToNearRatio}.
   * </p>
   *
   * @type {Number}
   * @default 1e9
   */
this.logarithmicDepthFarToNearRatio = 1e9;
```
&ensp; &ensp; &ensp;对数深度的处理核心就是着色器当中的重新计算片元深度的代码，大家可以把深度计算公式在Graph中看一眼函数图像，一切都明白了！depth值越大，经过log之后曲线增长越缓，这说明什么？这说明片元离相机近截面越远，经过log算出来的depth值会比线性计算出来的值要小很多，值小，精度就够了！

-------------
文中如有错误，万望批评指正，邮箱:1780721345@qq.com

