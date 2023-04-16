---
layout: post
title:  "Fast Geodesic Distance Evaluation"
date:   2020-09-20 20:20:17 +0200
categories: geometry
---

If we have one point A on the surface of a mesh, the geodesic distance to another point B also on the surface of mesh, is the length of the shortest path joining A to B, while still walking always over the surface of the mesh. Joining all the points which are at the same geodesic distance from a point creates the isolines of the following imagen:

![Results](/assets/images/geodesic_result04.png){:class="img-responsive"}

Although some techniques already exists, I'm going to present my own algorithm which returns the exact geodesic distance. In runs on top of the half edge mesh structure, which defines an array of 3D vertices, edges and faces (not necessarily triangles, but convex polygons).

Rather than computing the distance from two arbitrary points A and B, we will solve the problem 'finding the shortest distance from vertex A to all other vertices in the mesh', and if we want to stop when we reach B we can stop calculations, as we don't know in advance which path we should follow to reach B from A.

The algorithm starts from the initial vertex A, and in each iteration will find the following nearest vertex. It maintains a list of potential candidates for the following 'nearest vertex' to speed up the algorithm.

First, we **define $$ d(V) $$ as the distance from vertex $$V$$ to the initial vertex $$A$$**. We know $$ d(A) = 0 $$ as it's the distance to itself.

The immediate next candidates are the vertices sharing an edge with $$A$$, where the distance is the length of the edge. Fine.

If $$A$$ happens to be a corner of a convex face of 4 or more vertices, then there is at least one vertex which we can't reach directly using the edges, but, as the face is planar and convex, we can keep computing the distance directly as the distance between the two vertices, and be confident that it is the shortest distance from the vertex to $$A$$. 

We can join both steps to a single one and say: **iterate over all vertices of all faces used by $$A$$**.

That was trivial!. Now the problem gets more interesting, $$A$$ is no longer used, but we have a new set of candidates ($$V_1 .. V_8$$) and their associated distances. Let's keep the list of candidates sorted by distance to $$A$$, by just inserting them in order every time we find one. Normally new vertices will be added at the end of the list because the way we build it, and in practice the list never becomes very large.

Then, while the list is empty:

1. Pick the first element from the list, let's call it $$V_1$$, it has the next shortest distance to $$A$$
2. Remove it from the list, and repeat what we did with $$A$$: Iterate over all vertices of all faces used by $$V_1$$. Some additional work is needed for each candidate $$B_1$$:

    * If the vertex $$B_1$$ was already processed, skip the candidate, we already know the shortest distance to A. In the previous example, $$V_7$$ will iterate over $$V_8$$ and $$V_6$$, skip those candidates.

    * The distance associated to $$B_1$$ is the distance already associated to $$A_1$$ plus the distance from $$A_1$$ to $$B_1$$. **This is not correct but we will fix it shortly**.

    * If the connected vertex $$B_1$$ was not already in the list, just add it using the distance $$B_1$$

    * If the connected vertex $$B_1$$ was already in the list, but the new distance is smaller, update the distance associated to $$B_1$$ and update its ranking in the list, it might have won some positions.

We have required two new tests:

1. Check if the vertex is still in the list
2. Check if the vertex was already processed, so if it was ever added to the list.

I implement those tests using two bits arrays, indexed using the vertex ID. Start clearing all bits, and set to 1 every time we add the vertex to the list for the first condition, and once we remove the vertex from the list for the second.

If we upload the array of distances to the GPU, each vertex in the vertex shader can read it's own distance, send it to the pixel shader, and in the pixel shader use something similar to the following to render some nice bands:

```hlsl
  float clr = frac( distance );
  clr = 1 - pow(x,40);
```

The idea is that the distance goes from to 0 to N units, we want to render a black band each time we cross an integer value. So, take the decimal part, which goes from 0..1, compute `pow(x,40)` (or some arbitrary big number), only the values close to 1 will remain, all the others become almost zero. The last step just swaps black and white so we end with almost white and some thin black lines near the end of the decimal values.

If we follow the previous algorithm with a simple grid, starting from one corner and use the previous shader we get something like:

![Results](/assets/images/geodesic_00.png){:class="img-responsive"}

Opps!, as you can see, the distances are not correct!!, in such simple mesh the correct answer are some circular rings around the corner A.

In the following image we can see the problem on a simple two faces mesh.

![Results](/assets/images/geodesic_02.png){:class="img-responsive"}

The shortest distance from A to D is not obtained by going through B or C, but by going straight from A to D. But A is not in the same polygon!!, probably not even the same plane!! and later on, the original A is going to be far away. We want to compute the distance to A on each vertex based on the computations from previous near vertices. So the problem becomes: 

If we know the distance to A at vertices B and C, but we don't know where is A, can we compute exactly the distance A to D?

What we are going to do is find an alternative A', and solve the problem in a single plane. Recall d(V) is the distance from any vertex V to A, and we know d(B) and d(C).
1. On the plane defined by vertices B, C and D, create a circle of radius d(B) centered at B, 
2. Add another circle of radius d(C) centered at C. 
3. These two circles intersect at two points in the plane BCD. Both defines a potencial position A', that makes B and C have their distances, the other is the same position reflected around the line defined by B and C. 

![Results](/assets/images/geodesic_04.png){:class="img-responsive"}

We are interested in the solution that is not in the same side as D, as per construction the original A is far from D than it is from B or C. Also, this A' is not the original A, it's an equivalent position on the plane BCD that we can use to find if a shorter distance between the original A and D exists. 

Now, if the segment that goes from A' to D crosses the segment CB, the distance A'D is accepted as a shorter path to go from the original A to D without passing by B or C. Recall that we have not used the position from A, we have just created an imaginary A' to solve the problem in the plane BCD!

## Solving in 2D

Another good point is that because we are in a plane, we can solve the problem in 2D. In fact, to simplify I like to set B in the origin, and scale C so it's at (0,1). I will use lowercase names for the new coordinate system. $$b=(0,0)$$ $$c=(0,1)$$

$$r_b = {d(B) \over length(BC)}$$

$$r_c = {d(C) \over length(BC)}$$

$$x^2 + y^2 = r_b^2$$

$$x^2 + (y-1)^2 = r_c^2$$

In this coordinate system, $$a = (x,y)$$ is the solution to the previous equations.

$$y^2 - (y-1)^2 = r_b^2 - r_c^2$$

$$y^2 - (y^2 + 1 - 2y) = r_b^2 - r_c^2$$

and then find if x exists:

$$x = \pm\sqrt{r_b^2 - y^2}$$

Transform D to this 2D coordinate system and call it $d$. 
* Y = in the unit vector from B to C
* Z = The Normal of the plane BCD
* X = cross(Y,Z)
  $$d = (dot(X,D-B),dot(Y,D-B))$$
Remember that we have two solutions for x, one for each sign of the root.

Now we need to check if the segment from $$a$$ to $$d$$ intersects the segment $$b$$ to $$c$$. If we express the segment as:

$$r(t) = a + t * (d - a)$$

The intersection with segment $$b$$ to $$c$$ happens when 

$$r_x(t) = 0$$

$$t = {-a_x \over {d_x - a_x }}$$
 
Check if at the intersection point, the y is in the correct range, between 0 and 1.

$$r_y = a_x + t * ( d_x - a_x )$$

Because between the two coordinate system there is just a scale and rotation, the distance in the 3D system is the distance in 2D scaled by the factor $$length(BC)$$. So the distance between A' and D is:
$$length(d - c) * length(BC)$$

## Back to 3D

What happens if the segment A'D passes through B (or C)? It means the solution is exactly the sum of d(B) and the distance between B and D.
And if it crosses outside the segment B and C? In this case there must be another segment from B or C or other vertices for which we already know the distance to A that we can use to compute the distance to the new candidate D.

Back to the algorithm, start from the wrong approximation, and check if using the vertices connected to $$B_1$$ for which we already know the distance to A, we can use the *two circle method* to find a potentially shorter distance. If found it, use it as key in the ordered list of future candidates.

This is the result with the distance properly calculated.

![My image Name](/assets/images/geodesic_01.png){:class="img-responsive"}

There is a visible artifact inside the first face close to A. This is due to pixel shader interpolation, as I'm rendering two triangles per quad, and I'm evaluating the distance at each vertex.

## Summary

To need to find the distance from A to D in this mesh on the left, we instead solve the equivalent problem in the mesh on the right. We are not rotating the plane, we reconstruct a position A' in the plane BCD and knowing the d(B) and d(C) find d(D)

![My image Name](/assets/images/geodesic_05.png){:class="img-responsive"}

## How fast is this algorithm?

Assuming you already have the mesh stored in the half mesh format, visiting the neighborhood vertices is O(1). We need to maintain a list of candidates sorted by distance, but the list it's not going to be larger than the maximum number of vertices that are at the same distance of the original vertex A. Each test requires some computations and square roots, but nothing very complicated.

Some timings for 3 different models for a single thread

| Torus   | Vertices |    Faces|   Edges | Time| 
|    ---: |    ---: |  ---: |  ---: |---: |
| Torus   | 576    |      576 |     2304 | 0.001 s| 
| Fandisk |  7.000 |   14.000 |   43.000 | 0.011 s| 
| Bunny   | 55.000 |  111.000 |  334.000 | 0.091 s| 

## Results

Now the mandatory images of the results:

![My image Name](/assets/images/geodesic_result00.png){:class="img-responsive"}
![My image Name](/assets/images/geodesic_result01.png){:class="img-responsive"}
![My image Name](/assets/images/geodesic_result02.png){:class="img-responsive"}
![My image Name](/assets/images/geodesic_result03.png){:class="img-responsive"}
![My image Name](/assets/images/geodesic_result04.png){:class="img-responsive"}
