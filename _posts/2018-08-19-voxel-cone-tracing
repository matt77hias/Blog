---
layout: post
title:  "Voxel Cone Tracing"
date:   2018-08-19
---

# Illumination components

If the scenes only contain point lights (e.g., omni lights, spotlights, etc.) and emissive surfaces, the illumination contributions at a surface position, $x$, are computed as follows:

 1. The self emission associated with the emissive surface (i.e. 0 bounces/surface interactions) is computed as usual (without using the voxelization; as is the case for adding this contribution to the scene's voxelization);
 2. The direct illumination associated with the point lights (i.e. 1 bounce/surface interaction) is computed as usual (without using the voxelization; as is the case for adding this contribution to the scene's voxelization);
 3. The direct illumination associated with the emissive surface (i.e. 1 bounce/surface interaction) is obtained from the scene's voxelization using voxel cone tracing;
 4. The indirect illumination associated with the point lights (i.e. 2 bounces/surface interactions) is obtained from the scene's voxelization using voxel cone tracing.

(By computing the diffuse illumination for all the voxels from the scene's voxelization using voxel cone tracing, the diffuse illumination from additional bounces/surface interactions can be accumulated inside the voxels. This requires each voxel to store (an approximation) to the normal distribution of the scene's surfaces as well overlapping those voxels.)

# Illumination from the scene's voxelization (3)-(4)

The outgoing radiance obtained from the scene's voxelization (subscript $v$) using ray tracing (subscript $r$):

$$L_o\!\left(x, \hat\omega_o\right) = \int_\Omega f_{r}\!\left(x, \hat\omega_o, \hat\omega_i\right) L_{v,r}\!\left(x, \hat\omega_i\right) \left(\hat{n} \cdot \hat\omega_i\right) \mathrm{d}\hat\omega_i$$

For a diffuse BRDF $f_{r}\!\left(x, \hat\omega_o, \hat\omega_i\right) = \frac{k_d}{\pi}$:

$$L_o\!\left(x, \hat\omega_o\right) = \frac{k_d}{\pi} \int_\Omega L_{v,r}\!\left(x, \hat\omega_i\right) \left(\hat{n} \cdot \hat\omega_i\right) \mathrm{d}\hat\omega_i$$

For a partitioning of $\Omega$ in $N$ disjunct subdomains, $\Omega_j$, (e.g., $\approx$ cones) (i.e. $\Omega_1 \cup~...~\cup \Omega_N = \Omega$ with $\Omega_i \cap \Omega_j = \emptyset$ for each $i \ne j$):

$$L_o\!\left(x, \hat\omega_o\right) = \frac{k_d}{\pi} \sum_{j = 1}^{N} \int_{\Omega_{j}}  L_{v,r}\!\left(x, \hat\omega_{j,i}\right) \left(\hat{n} \cdot \hat\omega_{j,i}\right) \mathrm{d}\hat\omega_{j,i}$$

Relying on a single cone (subscript $c$) with a direction, $\hat\omega_{j}$, instead of individual rays for each subdomain, $\Omega_j$: 

$$L_o\!\left(x, \hat\omega_o\right) \approx \frac{k_d}{\pi} \sum_{j = 1}^{N} L_{v,c}\!\left(x, \hat\omega_{j}\right) \int_{\Omega_{j}} \left(\hat{n} \cdot \hat\omega_{j,i}\right) \mathrm{d}\hat\omega_{j,i}$$

$$W_j = \int_{\Omega_{j}} \left(\hat{n} \cdot \hat\omega_{j,i}\right) \mathrm{d}\hat\omega_{j,i}$$

$$\hat{W}_j = \frac{W_j}{\pi}$$

$$L_o\!\left(x, \hat\omega_o\right) \approx k_d \sum_{j = 1}^{N} \hat{W}_j L_{v,c}\!\left(x, \hat\omega_{j}\right)$$

We can use for example the following six cones, each with an aperture of $\frac{\pi}{6}$:

[![enter image description here][1]][1]

The normalized weight of the blue cone about the surface normal is equal to:

$$\hat{W}_{\mathrm{blue}} = \frac{1}{\pi} \int_{0}^{2\pi} \int_{0}^{\frac{\pi}{6}} \cos\!\theta \sin\!\theta \, \mathrm{d}\theta \, \mathrm{d}\phi = \frac{1}{4}$$

The normalized weight of the other cones is approximately equal to:

$$\hat{W}_{\mathrm{purple|red|green|orange|brown}} \approx \frac{1-\hat{W}_{\mathrm{blue}}}{5} = \frac{3}{20}$$

For a specular BRDF, one typically uses a single cone with a direction, $2 \left(\hat{n} \cdot \hat\omega_o\right) \hat{n}-\hat\omega_o$ (i.e. reflected direction of $\hat\omega_o$ about $\hat{n}$, and an aperture based on the roughness of the surface material.

# Voxel Cone Tracing

$L_{v,c}$ is computed using voxel cone tracing, which can be implemented by marching the mip-mapped 3D voxel texture in UVW texture space.

Our accumulated radiance (red, green, blue channels) and opacity (alpha channel) are initialized to zero. 

    Lv ← [0, 0, 0, 0]

Our cone marching distance is initialized to an offset to avoid sampling the self emission associated with the emissive surface and the direct illumination associated with the point lights twice. We need to basically skip the voxel containing the surface position for which the outgoing radiance needs to be estimated.

    distance ← voxel_offset

Marching continues until we reach an accumulated opacity of 1 or more:

    while Lv.alpha < 1:
    	compute diameter (i.e. cone voxel footprint)
    	compute mip_level
    	compute position at distance
    	if mip_level >= max_mip_level or position ∉ [0,1]^3
    	    break
    	sample Lv_k(position, mip_level)
    	Lv ← Lv + (1-Lv.alpha) * Lv_k   // blend equation
    	distance ← distance + cone_step // marching -> diameter and mip_level will increase
    return Lv

[![enter image description here][2]][2]


  [1]: https://i.stack.imgur.com/TbALB.png
  [2]: https://i.stack.imgur.com/dGx7V.png
