---
layout: post
title:  "Creating a view frustum in local/world/camera space using SIMD"
date:   2017-08-24
categories: culling
---

$$\begin{align}
\mathrm{p_{p}} 
&= \left( x_{\mathrm{p}}, y_{\mathrm{p}}, z_{\mathrm{p}}, w_{\mathrm{p}} \right) \\
&= \left( x_{s}, y_{s}, z_{s}, w_{s} \right) \mathrm{T}_{i \rightarrow \mathrm{p}} \\
&= \mathrm{p}_{s} \mathrm{T}_{i \rightarrow \mathrm{p}},
\end{align}$$
with $$s \in \{\mathrm{l}, \mathrm{w}, \mathrm{c}\}$$.

$$\begin{align}
\mathrm{p_{ndc}} 
&= \left( x_{\mathrm{ndc}}, y_{\mathrm{ndc}}, z_{\mathrm{ndc}}, w_{\mathrm{ndc}} \right) \\
&= \mathrm{p_{p}} / w_{\mathrm{p}} \\
&= \left( x_{\mathrm{p}}/w_{\mathrm{p}}, y_{\mathrm{p}}/w_{\mathrm{p}}, z_{\mathrm{p}}/w_{\mathrm{p}}, 1 \right).
\end{align}$$

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
