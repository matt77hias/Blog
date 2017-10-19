---
layout: post
title:  "NDC to View space"
date:   2017-09-07
categories: transformation
---

# Introduction
In [deferred shading](https://en.wikipedia.org/wiki/Deferred_shading), geometrical (normal, depth) and material data is stored in a GBuffer in a first pass. 
The actual lighting takes place in a second pass based on the data stored in the GBuffer.
With regard to geometrical data, we minimally need a surface position and normal, both expressed in camera view or world space coordinates depending on the space used for lighting calculations. 
There is no need, however, for storing an explicit surface position in the GBuffer (and thus wasting valuable memory resources and bandwidth), since this surface position can be reconstructed.
(*For the remainder, we assume that the lighting calculations take place in camera view space. 
If you want to use world space instead, you need to additionally transform the surface position and normal from camera view to world space before applying your lighting calculations.*)

## Perspective Camera Only Approach
A (row-major) perspective transformation matrix has the following format:

$$\begin{align} 
\mathrm{T}^{\mathrm{v \rightarrow p}}
&= \begin{bmatrix} 
\mathrm{T}_{00} &0 &0 &0 \\ 
0 &\mathrm{T}_{11} &0 &0 \\ 
0 &0 &\mathrm{T}_{22} &1 \\ 
0 &0 &\mathrm{T}_{32} &0 \end{bmatrix}
=\begin{bmatrix} 
1/x &0 &0 &0 \\ 
0 &1/y &0 &0 \\ 
0 &0 &-w &1 \\ 
0 &0 &z &0 \end{bmatrix} \! . 
\end{align}$$

This transformation matrix is used to transform (*homogeneous*) points from view ($$p^{\mathrm{v}}$$) to projection ($$p^{\mathrm{p}}$$) space, after which the homogeneous divide ($$p_{w}^{\mathrm{p}}$$) is applied to transform to NDC (Normalized Device Coordinate, $$p^{\mathrm{ndc}}$$) space. (*NDC space is technically a 3D space, but for ease of notation, I use 4D points with a $$w=1$$*). If we explicitly write down this chain of transformations, we obtain:

$$\begin{align}
p^{\mathrm{v}} \mathrm{T}^{\mathrm{v} \rightarrow \mathrm{p}}  &= \left(\frac{1}{x} p_{x}^\mathrm{v}, \frac{1}{y} p_{y}^\mathrm{v}, -w~p_{z}^\mathrm{v} + z, p_{z}^\mathrm{v}\right) = p^{\mathrm{p}} \\
p^{\mathrm{p}}/p_{w}^{\mathrm{p}} &= \left(\frac{1}{x} \frac{p_{x}^\mathrm{v}}{p_{z}^\mathrm{v}}, \frac{1}{y} \frac{p_{y}^\mathrm{v}}{p_{z}^\mathrm{v}}, -w + \frac{z}{p_{z}^\mathrm{v}}, 1\right) = p^{\mathrm{ndc}}.
\end{align}$$

In a deferred renderer, we need to go the other way around while resolving the GBuffer and could use four components ($$x$$, $$y$$, $$z$$, $$w$$, see above) to transform a point from NDC to view space:

$$\begin{align}
p_{z}^\mathrm{ndc} &= -w + \frac{z}{p_{z}^\mathrm{v}} 
&\Leftrightarrow p_{z}^\mathrm{v} &= \frac{z}{p_{z}^\mathrm{ndc} + w}\\
p_{x}^\mathrm{ndc} &= \frac{1}{x} \frac{p_{x}^\mathrm{v}}{p_{z}^\mathrm{v}} 
&\Leftrightarrow p_{x}^\mathrm{v} &= x~p_{x}^\mathrm{ndc}~p_{z}^\mathrm{v}\\
p_{y}^\mathrm{ndc} &= \frac{1}{y} \frac{p_{y}^\mathrm{v}}{p_{z}^\mathrm{v}} 
&\Leftrightarrow p_{y}^\mathrm{v} &= y~p_{y}^\mathrm{ndc}~p_{z}^\mathrm{v}.
\end{align}$$

```c++
/**
 Returns the projection values from the given projection matrix to construct 
 the view position coordinates from the NDC position coordinates.

 @return   The projection values from the given projection matrix to 
           construct the view position coordinates from the NDC position 
           coordinates.
 */
inline const XMVECTOR XM_CALLCONV GetViewPositionConstructionValues(
    FXMMATRIX projection_matrix) noexcept {

    //        [ 1/X  0   0  0 ]
    // p_view [  0  1/Y  0  0 ] = [p_view.x 1/X, p_view.y 1/Y, p_view.z (-W) + Z, p_view.z] = p_proj
    //        [  0   0  -W  1 ]
    //        [  0   0   Z  0 ]
    //
    // p_proj / p_proj.w        = [p_view.x/p_view.z 1/X, p_view.y/p_view.z 1/Y, -W + Z/p_view.z, 1] = p_ndc
    //
    // Construction of p_view from p_ndc and projection values
    // 1) p_ndc.z = -W + Z/p_view.z       <=> p_view.z = Z / (p_ndc.z + W)
    // 2) p_ndc.x = p_view.x/p_view.z 1/X <=> p_view.x = X * p_ndc.x * p_view.z
    // 3) p_ndc.y = p_view.y/p_view.z 1/Y <=> p_view.y = Y * p_ndc.y * p_view.z

    const F32 x = 1.0f / XMVectorGetX(projection_matrix.r[0]);
    const F32 y = 1.0f / XMVectorGetY(projection_matrix.r[1]);
    const F32 z =  XMVectorGetZ(projection_matrix.r[3]);
    const F32 w = -XMVectorGetZ(projection_matrix.r[2]);

    return XMVectorSet(x, y, z, w);
}
```

# Orthographic Camera Only Approach
An (row-major) orthographic transformation matrix has the following format:

$$\begin{align} \mathrm{T}^{\mathrm{v \rightarrow p}}
&= \begin{bmatrix} 
\mathrm{T}_{00} &0 &0 &0 \\ 
0 &\mathrm{T}_{11} &0 &0 \\ 
0 &0 &\mathrm{T}_{22} &0 \\ 
0 &0 &\mathrm{T}_{32} &1 \end{bmatrix}
=\begin{bmatrix} 
1/x &0 &0 &0 \\ 
0 &1/y &0 &0 \\ 
0 &0 &1/z &0 \\ 
0 &0 &-w &1 \end{bmatrix} \! . 
\end{align}$$

This transformation matrix is used to transform (*homogeneous*) points from view ($$p^{\mathrm{v}}$$) to projection ($$p^{\mathrm{p}}$$) space, after which the homogeneous divide (no-op) is applied to transform to NDC (= projection) space. (*So basically a non-uniform scaling followed by a translation of the z component*).

If we explicitly write down this chain of transformations, we obtain:

$$\begin{align}
p^{\mathrm{v}} \mathrm{T}^{\mathrm{v} \rightarrow \mathrm{p}}  &= \left(\frac{1}{x} p_{x}^\mathrm{v}, \frac{1}{y} p_{y}^\mathrm{v}, \frac{1}{z} p_{z}^\mathrm{v} - w, 1\right) = p^{\mathrm{p}} = p^{\mathrm{ndc}}.
\end{align}$$

Again, we could use four components ($$x$$, $$y$$, $$z$$, $$w$$, see above) to transform a point from NDC to view space:

$$\begin{align}
p_{z}^\mathrm{ndc} &= \frac{1}{z} p_{z}^\mathrm{v} - w
&\Leftrightarrow p_{z}^\mathrm{v} &= z~\left(p_{z}^\mathrm{ndc} + w\right) \\
p_{x}^\mathrm{ndc} &= \frac{1}{x} p_{x}^\mathrm{v} 
&\Leftrightarrow p_{x}^\mathrm{v} &= x~p_{x}^\mathrm{ndc} \\
p_{y}^\mathrm{ndc} &= \frac{1}{y} p_{y}^\mathrm{v} 
&\Leftrightarrow p_{y}^\mathrm{v} &= y~p_{y}^\mathrm{ndc}.
\end{align}$$

```c++
/**
 Returns the projection values from the given projection matrix to construct 
 the view position coordinates from the NDC position coordinates.

 @return   The projection values from the given projection matrix to 
           construct the view position coordinates from the NDC position 
           coordinates.
 */
inline const XMVECTOR XM_CALLCONV GetViewPositionConstructionValues(
    FXMMATRIX projection_matrix) noexcept {

    //        [ 1/X  0   0  0 ]
    // p_view [  0  1/Y  0  0 ] = [p_view.x 1/X, p_view.y 1/Y, p_view.z 1/Z -W, 1] = p_proj = p_ndc
    //        [  0   0  1/Z 0 ]
    //        [  0   0  -W  1 ]
    //
    // Construction of p_view from p_ndc and projection values
    // 1) p_ndc.z = p_view.z/Z -W <=> p_view.z = Z * (p_ndc.z + W)
    // 2) p_ndc.x = p_view.x/X    <=> p_view.x = X * p_ndc.x
    // 3) p_ndc.y = p_view.y/Y    <=> p_view.y = Y * p_ndc.y

    const F32 x = 1.0f / XMVectorGetX(projection_matrix.r[0]);
    const F32 y = 1.0f / XMVectorGetY(projection_matrix.r[1]);
    const F32 z = 1.0f / XMVectorGetZ(projection_matrix.r[2]);
    const F32 w = -XMVectorGetZ(projection_matrix.r[3]);

    return XMVectorSet(x, y, z, w);
}
```

# Summary
So far, we have seen how to reconstruct the surface position in view space coordinates for a perspective and orthographic camera. Both reconstructions require only a `float4` coefficient vector in HLSL and are unfortunately quite different. So if our deferred renderer is going to support one camera type only, we could use the appropriate reconstruction. If our deferred renderer needs to support both or even more camera types, we need to specialize our shaders statically (i.e. pre-processor directives) or dynamically (i.e. dynamic branching based on some constant buffer flag) based on the camera type.

Alternatively, we can pass the inverse of our view-to-projection matrices (i.e. projection-to-view matrices) to reconstruct the view space coordinates from the NDC space coordinates.

$$\begin{align} 
\mathrm{T}^{\mathrm{v \rightarrow p}} &= \begin{bmatrix} 
\mathrm{T}_{00} &0 &0 &0 \\ 
0 &\mathrm{T}_{11} &0 &0 \\ 
0 &0 &\mathrm{T}_{22} &1 \\ 
0 &0 &\mathrm{T}_{32} &0 \end{bmatrix} \! , \\
\mathrm{T}^{\mathrm{p \rightarrow v}} &=\begin{bmatrix} 
1/\mathrm{T}_{00} &0 &0 &0 \\ 
0 &1/\mathrm{T}_{11} &0 &0 \\ 
0 &0 &0 &1/\mathrm{T}_{32} \\ 
0 &0 &1 &-\mathrm{T}_{22}/\mathrm{T}_{32}\end{bmatrix} \! . 
\end{align}$$

```c++
/**
 Returns the projection-to-view matrix of this perspective camera.

 @return   The projection-to-view matrix of this perspective 
           camera.
 */
virtual const XMMATRIX GetProjectionToViewMatrix() const noexcept override {
 const XMMATRIX view_to_projection = GetViewToProjectionMatrix();

 const F32 m00 = 1.0f / XMVectorGetX(view_to_projection.r[0]);
 const F32 m11 = 1.0f / XMVectorGetY(view_to_projection.r[1]);
 const F32 m23 = 1.0f / XMVectorGetZ(view_to_projection.r[3]);
 const F32 m33 = -XMVectorGetZ(view_to_projection.r[2]) * m23;

 return XMMATRIX {
   m00, 0.0f, 0.0f, 0.0f,
  0.0f,  m11, 0.0f, 0.0f,
  0.0f, 0.0f, 0.0f,  m23,
  0.0f, 0.0f, 1.0f,  m33
 };
}
```

**Orthographic Camera**

$$\begin{align} 
\mathrm{T}^{\mathrm{v \rightarrow p}} &= \begin{bmatrix} 
\mathrm{T}_{00} &0 &0 &0 \\ 
0 &\mathrm{T}_{11} &0 &0 \\ 
0 &0 &\mathrm{T}_{22} &0 \\ 
0 &0 &\mathrm{T}_{32} &1 \end{bmatrix} \! , \\
\mathrm{T}^{\mathrm{p \rightarrow v}} &=\begin{bmatrix} 
1/\mathrm{T}_{00} &0 &0 &0 \\ 
0 &1/\mathrm{T}_{11} &0 &0 \\ 
0 &0 &1/\mathrm{T}_{22} &0 \\ 
0 &0 &-\mathrm{T}_{32}/\mathrm{T}_{22} &1\end{bmatrix} \! . 
\end{align}$$

```c++
/**
 Returns the projection-to-view matrix of this orthographic camera.

 @return   The projection-to-view matrix of this orthographic 
           camera.
 */
virtual const XMMATRIX GetProjectionToViewMatrix() const noexcept override {
 const XMMATRIX view_to_projection = GetViewToProjectionMatrix();

 const F32 m00 = 1.0f / XMVectorGetX(view_to_projection.r[0]);
 const F32 m11 = 1.0f / XMVectorGetY(view_to_projection.r[1]);
 const F32 m22 = 1.0f / XMVectorGetZ(view_to_projection.r[2]);
 const F32 m32 = -XMVectorGetZ(view_to_projection.r[3]) * m22;

 return XMMATRIX {
   m00, 0.0f, 0.0f, 0.0f,
  0.0f,  m11, 0.0f, 0.0f,
  0.0f, 0.0f,  m22, 0.0f,
  0.0f, 0.0f,  m32, 1.0f
 };
}
```
