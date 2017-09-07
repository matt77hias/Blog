---
layout: post
title:  "NDC to View space"
date:   2017-09-07
categories: transformation
---

# Introduction
In [deferred shading](https://en.wikipedia.org/wiki/Deferred_shading), geometrical (normal, depth) and material data is stored in a GBuffer in a first pass.
The actual lighting takes place in a second pass based on the data stored in the GBuffer.
With regard to geometrical data, we typically need a surface position and normal. 
There is no need, however, for storing a surface position in the GBuffer (and thus wasting valuable memory resources), since this surface position can be reconstructed.

# The view-to-projection matrix

The view-to-projection matrix of a perspective camera has the following format (assuming a row-major convention):

$$\begin{align}
\mathrm{T}^{\mathrm{v} \rightarrow \mathrm{p}} 
&= \begin{bmatrix} \mathrm{T}_{00} &0 &0 &0 \\ 0 &\mathrm{T}_{11} &0 &0 \\ 0 &0 &\mathrm{T}_{22} &1 \\ 0 &0 &\mathrm{T}_{32} &0\end{bmatrix} \\
&= \begin{bmatrix} \cot\left(\mathrm{\frac{fov_x}{2}}\right) &0 &0 &0 \\ 0 &\cot\left(\mathrm{\frac{fov_y}{2}}\right) &0 &0 \\ 0 &0 &\frac{\mathrm{z_f}}{\mathrm{z_f} - \mathrm{z_n}} &1 \\ 0 &0 &\frac{-\mathrm{z_n}\mathrm{z_f}}{\mathrm{z_f} - \mathrm{z_n}} &0\end{bmatrix}.
\end{align}$$

Here, $$\mathrm{fov_x}$$ and $$\mathrm{fov_y}$$ are the horizontal and vertical [field-of-view](https://en.wikipedia.org/wiki/Field_of_view_in_video_games), respectively, and $$\mathrm{z_n}$$ and $$\mathrm{z_f}$$ are the position of the near and far-plane of the [viewing frustum](https://en.wikipedia.org/wiki/Viewing_frustum), respectively.

Note that there are only four variable matrix entries which fully characterize the view-to-projection matrix: $$\mathrm{T}_{00}$$, $$\mathrm{T}_{11}$$, $$\mathrm{T}_{22}$$ and $$\mathrm{T}_{32}$$.
It is possible to pack these four values in a `float4` of a constant buffer to access them in HLSL. In order to avoid some divisions and additions in our HLSL code, we could use the following four projection values ($$x$$, $$y$$, $$z$$ and $$w$$) instead:

$$\begin{equation}
\mathrm{T}^{\mathrm{v} \rightarrow \mathrm{p}} 
= \begin{bmatrix} 1/x &0 &0 &0 \\ 0 &1/y &0 &0 \\ 0 &0 &-w &0 \\ 0 &0 &z &0\end{bmatrix}.
\end{equation}$$

**C++ Code**:
```c++
inline const XMVECTOR XM_CALLCONV GetProjectionValues(FXMMATRIX projection_matrix) noexcept {
    const float x = 1.0f / XMVectorGetX(projection_matrix.r[0]);
    const float y = 1.0f / XMVectorGetY(projection_matrix.r[1]);
    const float z =  XMVectorGetZ(projection_matrix.r[3]);
    const float w = -XMVectorGetZ(projection_matrix.r[2]);
    return XMVectorSet(x, y, z, w);
}
 ```

# Reconstructing the view position

Multiplying the position $$\mathrm{p^v}$$ in view space with the view-to-projection matrix results in the position $$\mathrm{p^p}$$ in projection space. 
Dividing the latter by its $$w$$ coordinate (i.e. the homogeneous divide), results in the position $$\mathrm{p^{ndc}}$$ in NDC space.
If we explicitly write down this chain of transformations, we obtain:

$$\begin{align}
\mathrm{p^v} \mathrm{T}^{\mathrm{v} \rightarrow \mathrm{p}}  &= \left(\frac{1}{x} \mathrm{p}_{x}^\mathrm{v}, \frac{1}{y} \mathrm{p}_{y}^\mathrm{v}, -w~\mathrm{p}_{z}^\mathrm{v} + z, \mathrm{p}_{z}^\mathrm{v}\right) = \mathrm{p^p} \\
\mathrm{p^p}/\mathrm{p}_{z}^\mathrm{p} &= \left(\frac{1}{x} \frac{\mathrm{p}_{x}^\mathrm{v}}{\mathrm{p}_{z}^\mathrm{v}}, \frac{1}{y} \frac{\mathrm{p}_{y}^\mathrm{v}}{\mathrm{p}_{z}^\mathrm{v}}, -w + \frac{z}{\mathrm{p}_{z}^\mathrm{v}}, 1\right) = \mathrm{p^{ndc}}.
\end{align}$$

Since the projection values ($$x$$, $$y$$, $$z$$ and $$w$$) and the position $$\mathrm{p^{ndc}}$$ in NDC space are known, we can reconstruct the position $$\mathrm{p^v}$$ in view space:

$$\begin{align}
\mathrm{p}_{z}^\mathrm{ndc} &= -w + \frac{z}{\mathrm{p}_{z}^\mathrm{v}} 
&\Leftrightarrow \mathrm{p}_{z}^\mathrm{v} &= \frac{z}{\mathrm{p}_{z}^\mathrm{ndc} + w}\\
\mathrm{p}_{x}^\mathrm{ndc} &= \frac{1}{x} \frac{\mathrm{p}_{x}^\mathrm{v}}{\mathrm{p}_{z}^\mathrm{v}} &\Leftrightarrow \mathrm{p}_{x}^\mathrm{v} &= x~\mathrm{p}_{x}^\mathrm{ndc}~\mathrm{p}_{z}^\mathrm{v}\\
\mathrm{p}_{y}^\mathrm{ndc} &= \frac{1}{y} \frac{\mathrm{p}_{y}^\mathrm{v}}{\mathrm{p}_{z}^\mathrm{v}} &\Leftrightarrow \mathrm{p}_{y}^\mathrm{v} &= y~\mathrm{p}_{y}^\mathrm{ndc}~\mathrm{p}_{z}^\mathrm{v}.
\end{align}$$

**HLSL Code:**
```c++
float NDCZtoViewZ(float p_ndc_z, float2 projection_values) {
    return projection_values.x / (p_ndc_z + projection_values.y);
}

float3 NDCtoView(float3 p_ndc, float4 projection_values) {
    const float p_view_z = NDCZtoViewZ(p_ndc.z, projection_values.zw);
    return float3(p_ndc.xy * projection_values.xy, 1.0f) * p_view_z;
}
```

# Notes

If the shading does not take place in view space, but in some other space (e.g. world-space), you need to reconstruct the position $$\mathrm{p^v}$$ in view space first. Next, you need to transform this position from view space to the space used for shading.

The approach does not work for orthographic cameras. The view-to-projection matrix for orthographic cameras is a just rescaling; $$\mathrm{p^{ndc}}=\mathrm{p^{p}}$$ (no homogeneous divide). Reconstructing the position in view space requires applying the inverse scaling to the position in NDC space.
