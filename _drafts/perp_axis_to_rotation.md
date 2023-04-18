---
layout: post
title:  "Perpendicular Direction to Quaternion Rotation"
date: 2023-04-16 20:20:17 +0200
categories: geometry algorithm
---

I find quaternions to be incredibly convenient when it comes to rotating points in arbitrary directions. All you need to do is define an axis vector (represented by $$\vec{v}$$ ) and an angle of rotation, and then build a 4-component vector using this information:

$$ \vec{q} =[ { sin( { angle \over 2 })} * \vec{v}, cos( { angle \over 2 })] $$

I recently required to know in which direction I was rotating a point. I mean, I have a point $$\vec{p}$$, I rotate the point using the quaternion $$\vec{q}$$, and the point moves to the new position $$\vec{r}$$. To get it, I was rotating again the point by a small quaternion with the same rotation axis $$\vec{v}$$ and a very small angle, substracting those points to get the direction.

Later on, I realized there is an much simpler and precise way to obtain this direction. All we need to do is perform a cross product between the quaternion axis $$\vec{v}$$ and the point we want to rotate. This will give us the perpendicular direction to the rotation axis. 

$$ \vec{f} = \vec{ v } \times \vec{p} $$

Remember to normalize the result.

In my c++ code:

```c++

    VEC3 local_forward = VEC3( q.x, q.y, q.z ).Cross( p ).Normalized();

```