---
title: Cesium-Precision 精度问题
date: 2020-11-08 12:47:06
tags:
---

``` bash

vec4 czm_translateRelativeToEye(vec3 high, vec3 low)
{
    vec3 highDifference = high - czm_encodedCameraPositionMCHigh;
    vec3 lowDifference = low - czm_encodedCameraPositionMCLow;

    return vec4(highDifference + lowDifference, 1.0);
}

/**
* An automatic GLSL uniform representing a 4x4 model-view transformation matrix that
* transforms model coordinates, relative to the eye, to eye coordinates.  This is used
* in conjunction with {@link czm_translateRelativeToEye}.
*
* @example
* // GLSL declaration
* uniform mat4 czm_modelViewRelativeToEye;
*
* // Example
* attribute vec3 positionHigh;
* attribute vec3 positionLow;
*
* void main()
* {
*   vec4 p = czm_translateRelativeToEye(positionHigh, positionLow);
*   gl_Position = czm_projection * (czm_modelViewRelativeToEye * p);
* }
*
* @see czm_modelViewProjectionRelativeToEye
* @see czm_translateRelativeToEye
* @see EncodedCartesian3
*/
czm_modelViewRelativeToEye: new AutomaticUniform({
size: 1,
datatype: WebGLConstants.FLOAT_MAT4,
getValue: function (uniformState) {
    return uniformState.modelViewRelativeToEye;
},
}),
```

```bash
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

```bash
function EncodedCartesian3() {
  /**
   * The high bits for each component.  Bits 0 to 22 store the whole value.  Bits 23 to 31 are not used.
   *
   * @type {Cartesian3}
   * @default {@link Cartesian3.ZERO}
   */
  this.high = Cartesian3.clone(Cartesian3.ZERO);

  /**
   * The low bits for each component.  Bits 7 to 22 store the whole value, and bits 0 to 6 store the fraction.  Bits 23 to 31 are not used.
   *
   * @type {Cartesian3}
   * @default {@link Cartesian3.ZERO}
   */
  this.low = Cartesian3.clone(Cartesian3.ZERO);
}

/**
 * Encodes a 64-bit floating-point value as two floating-point values that, when converted to
 * 32-bit floating-point and added, approximate the original input.  The returned object
 * has <code>high</code> and <code>low</code> properties for the high and low bits, respectively.
 * <p>
 * The fixed-point encoding follows {@link http://help.agi.com/AGIComponents/html/BlogPrecisionsPrecisions.htm|Precisions, Precisions}.
 * </p>
 *
 * @param {Number} value The floating-point value to encode.
 * @param {Object} [result] The object onto which to store the result.
 * @returns {Object} The modified result parameter or a new instance if one was not provided.
 *
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

{% blockquote @superman 1780721345@qq.com %}
文章中有任何错误，请批评勘正
{% endblockquote %}
