---
layout: post
title:  "Creating a view frustum in local/world/camera space using SIMD"
date:   2017-08-24
categories: culling
---

$\mathrm{p_{proj}} = \left( x', y', z', w' \right) = \left( x, y, z, w \right) \mathrm{T}_{i \rightarrow \mathrm{proj}} = \mathrm{p}_{s}$ with $s \in \{\mathrm{local}, \mathrm{world}, \mathrm{camera}\} $

$\mathrm{p_{NDC}} = \left( x'', y'', z'', w'' \right) = \mathrm{p_{proj}} / w' = \left( x'/w', y'/w', z'/w', 1 \right)$

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
