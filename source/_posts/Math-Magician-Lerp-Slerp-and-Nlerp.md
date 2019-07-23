---
title: 'Math Magician - Lerp, Slerp, and Nlerp'
date: 2019-07-23 21:36:04
tags: [Unity, Math]
categories: GameEngine
---
 

# Math Magician – Lerp, Slerp, and Nlerp

[原文出处](https://keithmaggio.wordpress.com/2011/02/15/math-magician-lerp-slerp-and-nlerp/)


 ![Illustration of linear interpolation on a data...](http://blog.sensedevil.com/image/slerp1.svg.png)<div align="center"><small>*Illustration of linear interpolation on a data set. The same data set is used for other interpolation methods in the interpolation article. (Photo credit: Wikipedia)*</small></div>

Man, I wish I had this blog going through school, because this place has become my online notebook. Game development is like riding a bike – but math, for me, can have a hard time stickin’. I’ve been working in databases so long that I have a lot of [bit-shifting](http://en.wikipedia.org/wiki/Bitwise_operation "Bitwise operation") down, but my matrix and vector math is starting to lack. So, a creative way for me to remember all these algorithms is to try to explain them. Today, I’m going through [Linear Interpolation](http://en.wikipedia.org/wiki/Linear_interpolation "Linear interpolation"), Spherical Linear Interpolation, and Normalized Linear Interpolation.

<!-- more -->

**The Code**

I’m starting off with the code to show the similarities and differences with each. One thing similar right off the bat is the parameter list: each function takes the same 3 – A `start`, an `end`, and a `percent`. The `start` and `end` can be Vectors, Matrices, or Quaternions. For now, lets use Vectors. The `percent` is a scalar value between 0 and 1. If `percent == 0`, then the functions will return `start`, if `percent == 1`, the functions return `end`, and if `percent == 0.5`, it returns a nice average midpoint between the 2 vectors.

* * *

 ![English: Illustration of linear interpolation.](http://blog.sensedevil.com/image/slerp2.svg.png)<div align="center"><small>*Illustration of linear interpolation. (Photo credit: Wikipedia)*</small></div>

[**_Lerp:_**](http://en.wikipedia.org/wiki/Linear_interpolation "Linear interpolation") The basic Linear [Interpolation function](http://en.wikipedia.org/wiki/Sinc_function "Sinc function"). Lets break this down, right to left. First, we have `end - start`, which will pretty much return a vector the same distance as `start` and `end`, but represented as if it’s `start` vector was `{0, 0, 0}`. We multiply that vector by `percent`, which will give us a vector only `percent` as long (so if `percent` is 0.5, the vector now is only half as long as it was). We then add `start` back to it to return it to its normal position. Done.

```csharp
Vector3 Lerp(Vector3 start, Vector3 end, float percent)  
{  
     return (start + percent*(end - start));  
}
```

**Lerp is useful for:** Transitions from place to place over time, implementing a [Health Bar](http://en.wikipedia.org/wiki/Health_%28gaming%29 "Health (gaming)"), etc. Pretty much going from point A to point B.

* * *

 ![Oblique vector rectifies to Slerp factor.](http://blog.sensedevil.com/image/slerp3.png)<div align="center"><small>*Oblique vector rectifies to Slerp factor. (Photo credit: Wikipedia)*</small></div>

[**_Slerp:_**](http://en.wikipedia.org/wiki/Slerp "Slerp") Slerp is a very powerful function. It’s become very popular in the game industry for it’s ease of use and significant result, however, people tend to ignore its drawbacks. Slerp travels the _torque-minimal path_, which means it travels along the straightest path the rounded surface of a sphere, which is a major plus and part of the reason it’s so powerful, as well as the fact that it maintains _constant velocity_. But Slerp is _non-commutative_, meaning the order of how the vectors/matrices/quaternions are passed in will affect the result. A call to `Slerp(A, B, delta)` will yield a different result as compared to `Slerp(B, A, delta)`. Also, Slerp is computationally expensive because of the use of [mathematical functions](http://en.wikipedia.org/wiki/Function_%28mathematics%29 "Function (mathematics)") like [`Sin()`](http://en.wikipedia.org/wiki/Sine "Sine"), `Cos()` and `Acos()`.

```csharp
// Special Thanks to Johnathan, Shaun and Geof!  
Vector3 Slerp(Vector3 start, Vector3 end, float percent)  
{  
     // [Dot product](http://en.wikipedia.org/wiki/Dot_product) - the cosine of the angle between 2 vectors.  
     float dot = Vector3.Dot(start, end);  
     // Clamp it to be in the range of Acos()  
     // This may be unnecessary, but floating point  
     // precision can be a fickle mistress.  
     Mathf.Clamp(dot, -1.0f, 1.0f);  
     // Acos(dot) returns the angle between start and end,  
     // And multiplying that by percent returns the angle between  
     // start and the final result.  
     float theta = Mathf.Acos(dot)*percent;  
     Vector3 RelativeVec = end - start*dot;  
     RelativeVec.Normalize();  
     // [Orthonormal basis](http://en.wikipedia.org/wiki/Orthonormal_basis)  
     // The final result.  
     return ((start*Mathf.Cos(theta)) + (RelativeVec*Mathf.Sin(theta)));  
}
```
**Slerp is useful for:** Rotation, mostly.

* * *

 ![](http://blog.sensedevil.com/image/slerp4.gif)

**_Nlerp:_** Nlerp is our solution to Slerp’s computational cost. Nlerp also handles rotation and is much less computationally expensive, however it, too has it’s drawbacks. Both travel a torque-minimal path, but Nlerp _is_ commutative where Slerp is not, and Nlerp **_also_** does _not_ maintain a constant velocity, which, in some cases, may be a desired effect. Implementing Nlerp in place of some Slerp calls may produce the same effect and even save on some FPS. However, with every optimization, using this improperly may cause undesired effects. Nlerp should be used _more_, but it doesn’t mean cut out Slerp all together. Nlerp is very easy, too. Just normalize the result from `Lerp()`!

```csharp
Vector3 Nlerp(Vector3 start, Vector3 end, float percent)  
{  
     return Lerp(start,end,percent).normalized();  
}
```

**Nlerp is useful for:** Animation (in regards to rotation and such), optimized rotation.

* * *

The trouble with Lerp, Slerp and Nlerp to most people is that they don’t truly understand the functions. Most people will post “Hey, I have a problem with this” and someone will say use one form of Linear Interpolation, and just patch it in. It’ll work, but will they understand why it works? Could it be improved? I would recommend taking some time and reading articles on the different Linear Interpolations – there are more out there I didn’t mention, and all with different purposes.

That’s all for now. I’ll find more math to put up in the weeks to come. Till then, take care!

_This post is dedicated to the staff at [ZeeGee Games](http://www.zeegeegames.com/). I may have lost the battle, but I will not lose the war! I will be a Game Dev Jedi!_

_Also, if you’re part of the [IGDA](http://www.igda.org/ "International Game Developers Association"), [Vote For Grant! (This link is really old by now, though…)](http://grantforigda.wordpress.com/)_

References:  
[The Inner Product – Understanding Slerp, Then Not Using It, by Jonathan Blow](http://number-none.com/product/Understanding%20Slerp,%20Then%20Not%20Using%20It/)  
[GameDev.Net – Nlerp/Slerp with 2 vectors. \[Solved\], Post by ChaosPhoenix, Reply from jyk](http://www.gamedev.net/topic/543189-nlerpslerp-with-2-vectors-solved/)
