---
title: Weighted Vertex Normals
date: 2022-04-28 09:49:04
tags: normal
categories: Graphics
---
# Weighted Vertex Normals

* * *

[原文链接](https://www.bytehazard.com/articles/vertnorm.html)
 
### 3DSMAX Plugin

A popular MaxScript plugin for 3DSMAX (previously hosted on GitHub) is now available [here](https://www.bytehazard.com/articles/wnormals.html).

## Introduction

When rendering 3D triangle mesh geometry, vertex normal vectors ('_normals_') need to be computed either in real-time or at design-time, to achieve proper lighting of curved surfaces. There are a few ways to do this, however, the most commonly used method has significant flaws. This article illustrates two of those problems, and proposes a practical and robust solution.

The assumption is made that the reader is familiar with [surface normal vectors](http://en.wikipedia.org/wiki/Surface_normal) and their [application](http://en.wikipedia.org/wiki/Phong_shading) in 3D graphics rendering.

### Notes

Geometry may have 'hard edges' (sometimes called 'sharp edges'), in which case multiple vertex normals may lie at the same vertex position. This is covered here for completeness sake only; the proposed enhancements do not have any effect on the presence or appearance of hard edges.

The included code uses per-polygon smoothing-groups (most popular in 3D content authoring software) to achieve hard edges, though this can be substituted by any (algorithmic) criteria (e.g. angle between surface normals greater than *x* degrees).

Hard edges can also be created by duplicating vertices (for optimal rendering on modern graphics hardware), but for simplicity we'll assume there are no duplicate vertices present in the model.

Also note that the pseudo-code presented herein is far from optimal, and can be optimized in many ways.

<!-- more -->

### Terminology

| Parameter | Desciption |
|-----------|------------|
| face | A triangle consisting of three vertices. |
| polygon | A planar surface consisting of three or more vertices. |
| facet normal | The normal vector of the plane in which a face or polygon lies. |
| vertex normal | A normal at one of the three vertices of a face. There may be more than one vertex normal per vertex position (hard edges). |


* * *

## The Problem

When vertex normals are generated, it generally goes like this:

```c++
for each face A in mesh
{
 n = face A facet normal
 
 // loop through all vertices of face A
 for each vert in face A
 (
  for each face B in mesh
  {
   // ignore self
   if face A == face B then skip
   
   // criteria for hard-edges
   if face A and B smoothing groups match {
   
    // accumulate normal
    if faces share at least one vert {
     n += (face B facet normal)
    }
    
   }
  }
  
  // normalize vertex normal
  vn = normalize(n)
 }
}
```

In English: vertex *v* will have a normal *n* which is the average of the combined facet normals of all connected polygons. In most situations this will look fine. But consider the situation below:

![vertnormals1](http://blog.sensedevil.com/image/vertnormals1.png)

In this case the triangles that make up the thin beveled edges of the box will 'claim' much of the normal orientation. This causes the 'rounded' shading on the large flat sides of the box (problem #1).

This is then made worse because two corners of those large sides contribute 2x the facet normal (2 triangles touch the vertex), while the other two corners contribute only 1x (one triangle touches the vertex). This results into a discontinuity (the diagonal artifact in the above illustration) when shaded (problem #2).

A poor solution would be to simply align the normals to the axii of the faces when such geometry is generated (difficult to preserve) and/or to have an artist correct it by hand (labor intensive). However, we are interested in a generic solution that works for arbitrary geometry constructed from triangles (known as '_triangle soup_') and *n*\-sided polygons alike, requiring no artist intervention.

* * *

## Surface Area Weights

The solution is to determine the influence of each face in it's contribution to the vertex normal. The obvious way to do that is by using the surface area of each face as 'weight'. Small polygons will have little influence, large polygons have large influence.

```c++
for each face A in mesh
{
 n = face A facet normal
 
 // loop through all vertices in face A
 for each vert in face A
 {
  for each face B in mesh
  {
   // ignore self
   if face A == face B then skip
   
   // criteria for hard-edges
   if face A and B smoothing groups match {
   
    // accumulate normal
    if faces share at least one vert {
     n += (face B facet normal) * (face B surface area) // multiply by area
    }
    
   }
  }
  
  // normalize vertex normal
  vn = normalize(n)
 }
}
```

As you can see, we simply multiply the facet normal by the triangle area when we accumulate it. Since we are already normalizing the resulting vector, we don't have to do anything else. Behold:

![vertnormals2](http://blog.sensedevil.com/image/vertnormals2.png)

That looks much more pleasing. The beveled edges do actually still have have a slight influence over the larger sides of the box, but this is hardly noticeable in most situations (and in other cases even desirable).

* * *

## Angle Weights

While the above technique does fix the most visible problems, there's another issue that is worthwhile to consider. This problem is most noticeable on low-polygon cylindrical shapes:

![vertnormals3](http://blog.sensedevil.com/image/vertnormals3.png)

It may be hard to spot, but if you look closely you will notice a subtle shading discontinuity between vertex *A* and vertex *D*. The three faces that influence vertex *A*, are faces *t*, *u* and *v*. Although both faces *U* and *V* belong to the same polygon (*UV*), the facet normal of that polygon contributes twice. This causes the averaged vertex normal *A* to point slightly to our right.  
At vertex *C* the opposite happens, there *st* pulls the normal to our left. The result being that the two vertex normals diverge when they should be parallel.

We could potentially determine which faces lie in the same plane and skip accumulating coinciding facet normals, but this works only in a small number of situations.

A robust solution is to calculate the angle of the corners of the polygons, and use that as additional weight (just like surface area) at that corners vertex. In the above figure, at vertex *A*, the combined angles of the two corners of faces *u* and *v* will equal the corner angle of face *t*.

In pseudo code, that becomes:

```c++
for each face A in mesh
{
 n = face A facet normal
 
 // loop through all vertices in face A
 for each vert in face A
 {
  for each face B in mesh
  {
   // ignore self
   if face A == face B then skip
   
   // criteria for hard-edges
   if face A and B smoothing groups match {
   
    // accumulate normal
    // v1, v2, v3 are the vertices of face A
    if face B shares v1 {
     angle = angle_between_vectors( v1 - v2 , v1 - v3 )
     n += (face B facet normal) * (face B surface area) * angle // multiply by angle
    }
    if face B shares v2 {
     angle = angle_between_vectors( v2 - v1 , v2 - v3 )
     n += (face B facet normal) * (face B surface area) * angle // multiply by angle
    }
    if face B shares v3 {
     angle = angle_between_vectors( v3 - v1 , v3 - v2 )
     n += (face B facet normal) * (face B surface area) * angle // multiply by angle
    }
     
   }
  }
  
  // normalize vertex normal
  vn = normalize(n)
 }
}
```

Here, *angle* is the angle in radians (or degrees\*) between the two vectors of the two line segments that touch each of the three vertices in a face.

\* Because we normalize the end result, the angle may be computed/stored as either radians or degrees. Only the ratio between neighboring triangle features (surface area, corner angle) contributes as weight, so the choice of angular units does not matter.

* * *

## Conclusion

Weighted vertex normals improve the appearance of virtually all geometry, and is generally superior to the traditional non-weighted average. It works because the undesired shading artifacts are displaced from large (highly visible) polygons to their smaller neighbors (less visible), and thereby guarantees improved visuals in virtually all common situations.

Since vertex normal generation is most often a design-time process, there is no impact on performance, unless the normals are re-calculated from scratch in realtime.

Even though tangent space normal mapping is widely used nowadays, tangent vector use and computation requires the presence of vertex normals, and here these enhancements also improve visual quality. Tangent and bi-tangent vectors should be accumulated and weighted in parallel to vertex normals before orthogonalization for best quality.

Additionally, weighted vertex normals also allow for faux-rounded edges (smooth shaded beveled edges) without significantly increasing the polygon count, and can greatly reduce distortions of specular reflection highlights and environment mapped reflective/refractive surfaces.

* * *

Author: Martijn Buijs  
Created on: 2007-12-23  
Last modified: 2018-10-18  
Contact: martijn AT bytehazard DOT com

* * *

## 附录：3ds Max脚本

```MAXScript
-- WEIGHTEDNORMALS.MS

-- Computes Weighted Vertex Normals
-- by Martijn Buijs, 2014
-- www.bytehazard.com

-- 3DSMAX bugs encountered:
-- 1) .modifiers[#Edit_Normals] doesn't work on renamed modifiers for no
--     apparant reason
-- 2) We can only ever modify the topmost Edit_Normals modifier, even if we
--    properly access it through its handle. So if there's another Edit_Normals
--    modifier on the stack, we add a new modifier, so the user won't have his
--    changes overwritten.
-- 3) During testing, the script twice crashed on some geometry lacking
--    smoothing groups. Unable to reproduce.

global wnmodname = "Weighted Normals"


-- returns angle between two vectors
fn AngleBetweenVectors v1 v2 =
(
 return (acos (dot (normalize v1) (normalize v2) ) )
)


-- get weighted normals modifier
fn wnGetModifier obj =
(
 for i=1 to obj.modifiers.count do
 (
  local mf = obj.modifiers[i]
  if (classof mf) == Edit_Normals do
  (
   if (mf.name == wnmodname) then (
    return mf
   ) else (
    return undefined
   )
  )
 )
 return undefined
)


-- generates weighted normals
fn GenWeightedNormals obj =
(
 -- filter
 if (superClassOf obj) != GeometryClass do return false
 
 -- add mesh modifier
 if (classOf obj) != Editable_Mesh do
 (
  addModifier obj (Edit_Mesh())
 )
 
 -- detect existing modifier
 local mf = wnGetModifier obj
 
 -- modifier not found, create one
 if mf == undefined do
 (
  addModifier obj (Edit_Normals())
  mf = obj.modifiers[#Edit_Normals]
  mf.name = wnmodname
 )
 
 -- workaround for 3dsmax bug
 select obj
 max modify mode
  
 -- build face area array
 local facearea = #()
 facearea.count = obj.numFaces
 for i=1 to obj.numFaces do
 (
  facearea[i] = (meshop.getFaceArea obj i)
 )
 
 -- build face angle array
 local faceangle = #()
 faceangle.count = obj.numFaces
 for i=1 to obj.numFaces do
 (
  local f = getFace obj i
  local v1 = getVert obj f[1]
  local v2 = getVert obj f[2]
  local v3 = getVert obj f[3]
  local a1 = AngleBetweenVectors (v2-v1) (v3-v1) -- todo: optimize
  local a2 = AngleBetweenVectors (v1-v2) (v3-v2)
  local a3 = AngleBetweenVectors (v1-v3) (v2-v3)
  faceangle[i] = [a1,a2,a3]
 )
 
 -- get number of normals
 local normNum = mf.GetNumNormals()
 
 -- allocate array
 local norms = #()
 norms.count = normNum
 for i=1 to normNum do
 (
  norms[i] = [0,0,0]
 )
 
 -- loop faces
 for i=1 to obj.numFaces do
 (
  -- get face normal
  in coordsys local n = getFaceNormal obj i
  
  -- accumulate
  for j=1 to 3 do
  (
   local id = mf.GetNormalID i j
   norms[id] = norms[id] + (n * facearea[i] * faceangle[i][j])
  )
 )
 
 -- set normals
 for i=1 to normNum do
 (
  -- make explicit
  mf.SetNormalExplicit i explicit:true
  
  -- set normal vector
  mf.SetNormal i (normalize norms[i])
 )
)


--- GUI ------------------------------------------------------------------------


-- close existing floater
if WeightedNormals != undefined do
(
 closeRolloutFloater WeightedNormals
)

-- create floater
WeightedNormals = newRolloutFloater "Weighted Normals" 180 150


-- generate rollout
rollout rWNGenerate "Weighted Normals"
(
 button cmdCreate "Generate" width:140
 
 on cmdCreate pressed do
 (
  -- copy selection (can't copy arrays in 3dsmax)
  local sel = #()
  for i=1 to selection.count do
  (
   sel[i] = selection[i]
  )
  
  -- create selection list
  for i=1 to sel.count do
  (
   GenWeightedNormals sel[i]
  )
  
  -- restore selection
  selection = sel
 )
)
addRollout rWNGenerate WeightedNormals


-- about rollout
rollout rWNAbout "About"
(
 label lab1 "Weighted Normals 1.0.0"
 label lab2 "by Martijn Buijs"
 label lab3 "www.bytehazard.com"
)
addRollout rWNAbout WeightedNormals


-- END OF FILE
```
