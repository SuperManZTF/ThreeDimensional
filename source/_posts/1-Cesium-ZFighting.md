---
title: Cesium-Z-Fighting 深度冲突问题
---
&ensp; &ensp; &ensp; &ensp; 在Cesium引擎中深度冲突（Z-Fighting）是绕不开的。之所以会产生深度冲突，是因为两个表面过于接近，深度缓冲区有限的精度已经不能区分哪个在前哪个在后。解决深度冲突问题普遍的方法有“对数深度”和“多边形偏移”，在Cesium当中解决该问题有两种方式“多视锥体”和“多视锥体+对数深度”。
&ensp; &ensp; &ensp; &ensp; GIS-地球渲染的是行星级别的比例尺，用米度量的分辨率会出现深度冲突和抖动的现象。这里作者使用的是Cesium-1.71版本。关于深度冲突本质上是数值的精度问题，Cesium引擎中处理精度问题还有两种方式RTC和RTE，请结合《Cesium-Precision 精度问题》一文来理解。

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

&ensp; &ensp; &ensp; &ensp; 在顶点着色器中做了两个矩阵变换（本质上是三个MV乘起来了）：modelViewMatrix和projectionMatrix，将顶点从局部坐标系转到世界坐标系转到相机坐标系转到投影坐标系，这里的projectionMatrix矩阵是根据相机的视锥体构建的（构建过程参考ThreeJS-Camera），经过矩阵计算gl_Position是被转换到投影剪裁空间（具体参考冯乐乐《Unity Shader入门精要》第四章），此处只说结论：经过矩阵变换最终写入深度缓冲区的深度值是“片元距离相机视锥体近截面的距离”（这个结论如果理解不了就先暂时接受并大胆认可它）。

``` bash
// program
gl.getExtension('EXT_frag_depth');
// fragment shader
void main() {
  gl_FragColor = vec4(1.0, 0.0, 1.0, 1.0); 
  gl_FragDepthEXT = 0.5; 
}
```

&ensp; &ensp; &ensp; &ensp; 在WebGL shader中如果想在片元着色器中修改片元的深度需要用到WebGL扩展EXT_frag_depth，使用gl_FragDepthEXT可以修改片元的深度值，后边使用对数深度修改片元深度的时候会用到这个扩展，在Cesium中这个扩展及其重要，例如贴地线/贴地面等都会涉及到深度值的修改。

## 2. 多边形偏移

``` bash
// program
gl.enable(gl.POLYGON_OFFSET_FILL);// 启用多边形偏移
gl.polygonOffset(1.0, 1.0);
```

&ensp; &ensp; &ensp; &ensp; 开启多边形偏移之后，设置多边形偏移参数：gl.polygonOffset(factor, units), 指定加到每个顶点z值上的偏移量，偏移量按m*factor+r*units来计算，m表示顶点所在表面相对于相机视线的角度，r表示硬件能够区分两个z值之差的最小值。这种处理深度冲突的方式在深度值精度够用的情况下是可以的，但是解决不了精度不够用的情况。

## 3. 多视锥体渲染

&ensp; &ensp; &ensp; &ensp; 多视锥体渲染的关键是计算视锥体的远近截面和视锥体数量，在Cesium-view中的View.prototype.createPotentiallyVisibleSet方法。
