---
layout: post
title:  "Creating a view frustum in local/world/camera space using SIMD"
date:   2017-08-24
categories: culling
---

# GPU - View Frustum Culling
The fixed-function Rasterizer Stage (`RS`) of the graphics pipeline receives individual primitives as input and generates fragments as output. In order to generate these fragments, multiple functional operations are performed:
1. Primitive Culling
2. Primitive Clipping
3. Homogeneous Divide
4. Viewport Transformation
5. Fragment Generation
6. Scissor Test
7. Attribute Interpolation

The first operation, Primitive Culling, implies both back face culling (if enabled) and view frustum culling. From here on, we will only focus on the latter type of culling. 

If view frustum culling takes place after the homogeneous divide, the associated culling space corresponds to the `Normalized Device Coordinate` (NDC) space (denoted as $$\mathrm{n}$$). In this space, the following equations need to be satisfied:

$$\begin{align}
-1 &\le x_{\mathrm{n}} \le 1\\
-1 &\le y_{\mathrm{n}} \le 1\\
0 &\le z_{\mathrm{n}} \le 1
\end{align}$$

A point primitive is culled if its vertex does not satisfy these equations and thus is positioned outside the view frustum. A triangle primitive is culled if all three of its vertices do not satisy these equations.

By performing culling before the homogeneous divide, an expensive divide operation can be omitted for every culled primitive. The associated culling space corresponds to `projection space` (denoted as $$\mathrm{p}$$). In this space, the following equations need to be satisfied:

$$\begin{align}
-w_{\mathrm{p}} &\le x_{\mathrm{p}} \le w_{\mathrm{p}}\\
-w_{\mathrm{p}} &\le y_{\mathrm{p}} \le w_{\mathrm{p}}\\
0 &\le z_{\mathrm{p}} \le w_{\mathrm{p}}
\end{align}$$

We will use the same equations to perform culling outside of the graphics pipeline, on the CPU. This way we can decrease the number of draw calls and decrease the number of wasted (*the primitives will be culled anyway*) VS, DS, TS, HS and GS invocations on the GPU. Instead of culling individual primitives themselves, culling will be performed on a coarser (e.g. (sub)model) level. Furthermore, we can even cull entities which only have an associated volume, but no associated geometry (e.g. lights).

# CPU - View Frustum Culling
Typically, a `Bounding Volume` (BV) is associated with and expressed in the local space of each submodel. Since a submodel is completely contained in its BV, culling the BV is equivalent to culling the submodel: if the BV is completely positioned outside a view frustum, the contained submodel is also completely positioned outside that view frustum. All kinds of BVs can be used for this purpose as long as they fit the submodel tightly, and are cheap to cull against a view frustum. Note that the tightest BV of a submodel is the submodel itself, but this BV is in general very expensive to cull directly. Frequently used BVs include `Bounding Spheres` (BSs),  `Axis-Aligned Bounding Boxes` (AABBs) and `Oriented Bounding Boxes` (OBBs).

Now that we have BVs (the entities we are going to cull against the view frustum), all that remains is creating a view frustum in the local coordinate space of the BV and thus the local coordinate space of the submodel. A view frustum consists of six bounding planes. A plane $$\left(\hat{n} \vert d\right)$$ is mathematically characterized by a normal vector $$\hat{n}$$ (rotation) and a signed distance $$d$$ (translation) from the origin in the direction of its normal $$\hat{n}$$. 

A point $$p=\left(x,y,z\right)$$ satisfies the following relations:
* If $$\hat{n} \cdot p + d = 0$$, then $$p$$ lies on the plane $$\left(\hat{n} \vert d\right)$$.
* If $$\hat{n} \cdot p + d \gt 0$$, then $$p$$ lies above the plane $$\left(\hat{n} \vert d\right)$$.
* If $$\hat{n} \cdot p + d \lt 0$$, then $$p$$ lies below the plane $$\left(\hat{n} \vert d\right)$$.

Alternatively, a homogeneous point $$p=\left(x_{p},y_{p},z_{p},1\right)$$ satisfies the following relations:
* If $$\left(\hat{n}, d\right) \cdot p = 0$$, then $$p$$ lies on the plane $$\left(\hat{n} \vert d\right)$$.
* If $$\left(\hat{n}, d\right) \cdot p \gt 0$$, then $$p$$ lies above the plane $$\left(\hat{n} \vert d\right)$$.
* If $$\left(\hat{n}, d\right) \cdot p \lt 0$$, then $$p$$ lies below the plane $$\left(\hat{n} \vert d\right)$$.

If we use six inward facing planes for our view frustum, all points $$p$$ satisfying $$\left(\hat{n}, d\right) \cdot p \lt 0$$ for at least one plane of the view frustum will be culled.

But how do you obtain the six planes? Lets look at the transformation from $$s$$ (local, world, or camera) space to projection space:

$$\begin{align}
\mathrm{p_{p}} 
&= \left( x_{\mathrm{p}}, y_{\mathrm{p}}, z_{\mathrm{p}}, w_{\mathrm{p}} \right) \\
&= \left( x_{s}, y_{s}, z_{s}, 1 \right) \mathrm{T}_{i \rightarrow \mathrm{p}} \\
&= \mathrm{p}_{s} \mathrm{T}_{i \rightarrow \mathrm{p}}.
\end{align}$$

TODO

```c++
ViewFrustum::ViewFrustum(CXMMATRIX transform) {
	const XMMATRIX C = XMMatrixTranspose(transform);

	// Extract the view frustum planes from the given transform.
	// All view frustum planes are inward facing: 0 <= n . p + d

	// p' = (x',y',z',w') = (x,y,z,1) T = p T

	//   -w' <= x'
	// <=> 0 <= w' + x'
	// <=> 0 <= p . c4 + p . c1
	// <=> 0 <= p . (c4 + c1)
	m_left_plane = C.r[3] + C.r[0];
		
	//    x' <= w'
	// <=> 0 <= w' - x'
	// <=> 0 <= p . c4 - p . c1
	// <=> 0 <= p . (c4 - c1)
	m_right_plane = C.r[3] - C.r[0];
		
	//   -w' <= y'
	// <=> 0 <= w' + y'
	// <=> 0 <= p . c4 + p . c2
	// <=> 0 <= p . (c4 + c2)
	m_bottom_plane = C.r[3] + C.r[1];
		
	//    y' <= w'
	// <=> 0 <= w' - y'
	// <=> 0 <= p . c4 - p . c2
	// <=> 0 <= p . (c4 - c2)
	m_top_plane = C.r[3] - C.r[1];
		
	//     0 <= z'
	// <=> 0 <= p . c3
	m_near_plane = C.r[2];

	//    z' <= w'
	// <=> 0 <= w' - z'
	// <=> 0 <= p . c4 - p . c3
	// <=> 0 <= p . (c4 - c3)
	m_far_plane = C.r[3] - C.r[2];

	// Normalize the view frustum planes.
	for (size_t i = 0; i < 6; ++i) {
	    m_planes[i] = XMPlaneNormalize(m_planes[i]);
	}
}
```
