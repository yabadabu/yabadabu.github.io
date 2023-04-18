---
layout: post
title:  "Bend Operator"
date: 2023-04-16 20:20:17 +0200
categories: geometry algorithm
---

Long time ago I learned about the modifiers bend, twist and taper in some 3D packages. I was able to implement twist and taper easily, but I was unable to implement a proper bend, Surprisingly I'm unable to find a public explanation about how to implement it. Recently I succeeded so I decided to write about it, probably I can use it in the future when I forget the details.

The Bend operator takes a geometry, a line and an angle to bend the line. The mesh will follow the bend together with the line. I think it's a very intuitive tool but it's a bit tricky to implement. Here is a some shot.

<< Image >>

I like to start with a very easy case, with values instead of symbols, and then find the general solution. So we will analyze the problem in 2D, and then move to 3D.

## Solution in 2D

We have a segment, one endpoint is in the origin, the other is at L=2 units going up. I want to bend it $$90^o$$. Attached to the segment, is the mesh, which for this case it's going to be a tube.

<< Image initial condition >>  << Image end condition >>

We see that the point in the origin, at $$y=0$$ don't move at all.

We want the segment to bend $$90^o$$, and maintaining it's original length L. Than means, the arc becomes part of a circle of some radius r, with the center also at y=0.

The full length of any circumference of radius R is 2πR. If our segment bends for $$90^o$$, expressed in radians that's $$ π \over 2 $$, and we want that length to equal L. So:

$$ {π \over 2} * R = 2 $$

Or symbolically, 

$$ angle * R = L $$

From where we can conclude that: $$ R = {L \over angle} $$

Now, the bottom of the segment does not move, the top rotates what the user has requested ($$90^o$$) using radius R. Any position along the segment rotates using the same radius R, and angle which is proportional to their original position in the segment. A point at height $$ L \over 2$$, will rotate in our case $$45^o$$.

Now, what happens if a point is not exactly along the segment?, so the x is maybe 0.5? From the draw we can see that we can use the same formula to rotate an angle proportional to the height of the point, but the radius it's going to be smaller if the point is close to the center of the radius, and larger if the vertex is in the opposite side of the segment.

Putting all together:

Beware that if the angle = 0, radius goes to the infinity, meaning there is nothing to bend.

## Solution in 3D

Extending the previous example to 3D, let's keep the segment still going up, and call the direction of the segment $$S$$ .

1. The center of the circumference is still at distance R but the direction $$V$$ must be any direction perpendicular to the segment direction $$S$$. Maybe be axis $$X$$ as in the 2D example, maybe the axis $$Z$$. In general any direction perpendicular to $$S$$, we can define an angle yaw, that's is going to become another parameter for the user.

2. Following in our example, the vertex has a third component z, if fact it's the distance perpendicular to the axis $$S$$ and $$V$$, let's called $$W$$ and we can use a cross product to obtain that direction: $$W$$ = $$S \times V$$. 
Apply the same math as in 2D but expressed as rotations in a plane.

We want to bend the points inside the segment. What if a point is not in our segment of length L?

* If a point has $$y < 0$$, it should not be transformed.
* If a point has $$y > L$$, by a certain amount alpha = y - L. It should transform with the max angle (in our example $$ π \over 2 $$), but no more, and the extra alpha units become "move along the final direction"

In HLSL code:

```hlsl
// Parameters
float3 axis_origin;                 // The base of the segment
float3 axis_unit;                   // Normalized direction of the segment
float axis_length = length(axis);
float max_rotation;
float3 center_dir;

void applyBend( float3 P ) {
  // Move the sys coord to the axis_origin
  float3 delta = P - axis_origin;

  // Get the projection along the bend dir, between 0 and 1 if P is
  // in the bend range
  float normalized_height_p = dot( delta, axis_unit ) / axis_length;

  // when max_rotation = 0 > radius = inf, that's why the last line checks
  // this condition
  bool must_rotate = abs(max_rotation) > 0.0 && normalized_height_p >= 0;
  float radius = axis_length / max_rotation;
  float angle = saturate( normalized_height_p ) * max_rotation;
 
  // The center of rotation is perp to the axis defined at the radius
  float3 center = radius * center_dir;
  delta += center;

  if( normalized_height_p > 1 ) {
    // This controls the mesh after the rotation end point
    delta -= axis_length * axis_unit;
  }
  else {
    // In the case of the vertical axis, is equivalent to delta.y = 0
    delta -= dot( delta, axis ) * axis_unit;
  }

  // The real axis of rotation is perp to the bend direction and center dir
  float3x3 rot_matrix = mat33AngleAxis( -bend_axis, angle );
  float3 k = mul( rot_matrix, delta ) - center;

  return must_rotate ? (k + axis_origin) : P;
}

// Rotation with angle (in radians) and axis
float3x3 mat33AngleAxis(float3 axis, float angle)
{
    float c, s;
    sincos(angle, s, c);

    float t = 1 - c;
    float x = axis.x;
    float y = axis.y;
    float z = axis.z;

    return float3x3(
        t * x * x + c,      t * x * y - s * z,  t * x * z + s * y,
        t * x * y + s * z,  t * y * y + c,      t * y * z - s * x,
        t * x * z - s * y,  t * y * z + s * x,  t * z * z + c
    );
}
```

