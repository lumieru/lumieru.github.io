---
title: Unity的AnimationCurve的实现方法
date: 2019-05-24 21:02:15
tags: Unity
categories: GameEngine
---
# 原文

[原文出处](https://answers.unity.com/questions/464782/t-is-the-math-behind-animationcurveevaluate.html)

# 方法一：

没有把参数t从系数中分离开来，直接混在一起计算系数a，b，c，d。这样看算法比较直观，但是不是最优化的。

```csharp
float Evaluate(float t, Keyframe keyframe0, Keyframe keyframe1)
{
    float dt = keyframe1.time - keyframe0.time;

    float m0 = keyframe0.outTangent * dt;
    float m1 = keyframe1.inTangent * dt;

    float t2 = t * t;
    float t3 = t2 * t;

    float a = 2 * t3 - 3 * t2 + 1;
    float b = t3 - 2 * t2 + t;
    float c = t3 - t2;
    float d = -2 * t3 + 3 * t2;

    return a * keyframe0.value + b * m0 + c * m1 + d * keyframe1.value;
}
```

<!-- more -->

# 方法二：

通过解四元三次方程组，得到系数a，b，c，d，这样得出的系数计算公式看着很复杂，但是因为系数可以提前计算好，所以效率要比方法一好。

Unity define a curve with 2 keyframes, each composed of a point and a tangent. I guess the simplest curve matching that is a third degree polynomial (a cubic function). Given the 2 points and tangents, it is possible to compute the polynomial coefficients simply by solving the following equation system:

```csharp
(1)  a*p1x^3 + b*p1x^2 + c*p1x + d = p1y
(2)  a*p2x^3 + b*p2x^2 + c*p2x + d = p2y
(3)  3*a*p1x^2 + 2*b*p1x + c = tp1
(4)  3*a*p2x^2 + 2*b*p2x + c = tp2
```

You can solve this manually or using a computer algebra system.

This gives you:

```c#
float a = (p1x * tp1 + p1x * tp2 - p2x * tp1 - p2x * tp2 - 2 * p1y + 2 * p2y) / (p1x * p1x * p1x - p2x * p2x * p2x + 3 * p1x * p2x * p2x - 3 * p1x * p1x * p2x);
float b = ((-p1x * p1x * tp1 - 2 * p1x * p1x * tp2 + 2 * p2x * p2x * tp1 + p2x * p2x * tp2 - p1x * p2x * tp1 + p1x * p2x * tp2 + 3 * p1x * p1y - 3 * p1x * p2y + 3 * p1y * p2x - 3 * p2x * p2y) / (p1x * p1x * p1x - p2x * p2x * p2x + 3 * p1x * p2x * p2x - 3 * p1x * p1x * p2x));
float c = ((p1x * p1x * p1x * tp2 - p2x * p2x * p2x * tp1 - p1x * p2x * p2x * tp1 - 2 * p1x * p2x * p2x * tp2 + 2 * p1x * p1x * p2x * tp1 + p1x * p1x * p2x * tp2 - 6 * p1x * p1y * p2x + 6 * p1x * p2x * p2y) / (p1x * p1x * p1x - p2x * p2x * p2x + 3 * p1x * p2x * p2x - 3 * p1x * p1x * p2x));
float d = ((p1x * p2x * p2x * p2x * tp1 - p1x * p1x * p2x * p2x * tp1 + p1x * p1x * p2x * p2x * tp2 - p1x * p1x * p1x * p2x * tp2 - p1y * p2x * p2x * p2x + p1x * p1x * p1x * p2y + 3 * p1x * p1y * p2x * p2x - 3 * p1x * p1x * p2x * p2y) / (p1x * p1x * p1x - p2x * p2x * p2x + 3 * p1x * p2x * p2x - 3 * p1x * p1x * p2x));
```
Then, to evaluate the value:

```csharp
float Evaluate(float t)
{
    return a*t*t*t + b*t*t + c*t + d;
}
```

I checked with Unity with the following quick and dirty code:

```csharp
 using UnityEngine;
 [ExecuteInEditMode]
 public class TestAnimCurve : MonoBehaviour {
     public AnimationCurve anim = AnimationCurve.EaseInOut(0, 0, 1, 1);
     float a;
     float b;
     float c;
     float d;
     
     void Update () {
         float p1x= anim.keys[0].time;
         float p1y= anim.keys[0].value;
         float tp1=anim.keys[0].outTangent;
         float p2x=anim.keys[1].time;
         float p2y= anim.keys[1].value;
         float tp2= anim.keys[1].inTangent;
         Debug.Log(p1x+ ", " + p1y+ ", " + tp1 + ", " + p2x + ", " + p2y + ", " + tp2);
         Debug.Log("Evaluate Unity: " + anim.Evaluate(0.1f) + ", " + anim.Evaluate(0.2f) + ", " + anim.Evaluate(0.3f) + ", " + anim.Evaluate(0.4f) + ", " + anim.Evaluate(0.5f) + ", " + anim.Evaluate(0.6f) + ", " + anim.Evaluate(0.76f) + ", " + anim.Evaluate(0.88f) + ", " + anim.Evaluate(0.98f));
         a = (p1x * tp1 + p1x * tp2 - p2x * tp1 - p2x * tp2 - 2 * p1y + 2 * p2y) / (p1x * p1x * p1x - p2x * p2x * p2x + 3 * p1x * p2x * p2x - 3 * p1x * p1x * p2x);
         b = ((-p1x * p1x * tp1 - 2 * p1x * p1x * tp2 + 2 * p2x * p2x * tp1 + p2x * p2x * tp2 - p1x * p2x * tp1 + p1x * p2x * tp2 + 3 * p1x * p1y - 3 * p1x * p2y + 3 * p1y * p2x - 3 * p2x * p2y) / (p1x * p1x * p1x - p2x * p2x * p2x + 3 * p1x * p2x * p2x - 3 * p1x * p1x * p2x));
         c = ((p1x * p1x * p1x * tp2 - p2x * p2x * p2x * tp1 - p1x * p2x * p2x * tp1 - 2 * p1x * p2x * p2x * tp2 + 2 * p1x * p1x * p2x * tp1 + p1x * p1x * p2x * tp2 - 6 * p1x * p1y * p2x + 6 * p1x * p2x * p2y) / (p1x * p1x * p1x - p2x * p2x * p2x + 3 * p1x * p2x * p2x - 3 * p1x * p1x * p2x));
         d = ((p1x * p2x * p2x * p2x * tp1 - p1x * p1x * p2x * p2x * tp1 + p1x * p1x * p2x * p2x * tp2 - p1x * p1x * p1x * p2x * tp2 - p1y * p2x * p2x * p2x + p1x * p1x * p1x * p2y + 3 * p1x * p1y * p2x * p2x - 3 * p1x * p1x * p2x * p2y) / (p1x * p1x * p1x - p2x * p2x * p2x + 3 * p1x * p2x * p2x - 3 * p1x * p1x * p2x));
         Debug.Log("Evaluate Cubic: " + Evaluate(0.1f) + ", " + Evaluate(0.2f) + ", " + Evaluate(0.3f) + ", " + Evaluate(0.4f) + ", " + Evaluate(0.5f) + ", " + Evaluate(0.6f) + ", " + Evaluate(0.76f) + ", " + Evaluate(0.88f) + ", " + anim.Evaluate(0.98f));
     }
     
     float Evaluate(float t) {
         return a * t * t * t + b * t * t + c * t + d;
     }
   }
```

After modifing tangents of the animation curve, the debug messages produced by this code are:

```
 0, 0, -4.484611, 1, 1, -10.23884
 Evaluate Unity: -0.2431039, -0.1423873, 0.2018093, 0.6891449, 1.219279, 1.691871, 2.077879, 1.854902, 1.193726
 Evaluate Cubic: -0.2431039, -0.1423873, 0.2018092, 0.6891448, 1.219279, 1.691871, 2.077879, 1.854902, 1.193726
```

So it really seams that this approach is the math behind AnimationCurve.Evaluate ;)

# 方法三：

系数的计算公式看着比较简单，并且也可以提前计算好，效率应该和方法二差不多。

AnimationCurve is a simple spline of cubic equations, where for each segment you specify the (time, value) coordinates and slope for each of two end points (the first point's "out" "tangent", second one's "in"). A cubic equation is the simplest polynomial which can meet these criteria.

This is also known as a cubic Hermite spline, where the only difference is that AnimationCurve allows you to assign different slopes for each direction from a keyframe (a point), which creates discontinuities in the first derivative (sharp angles). However, this does not change the formula.

[https://en.wikipedia.org/wiki/Cubic\_Hermite\_spline](https://en.wikipedia.org/wiki/Cubic_Hermite_spline)

It took me some time (much more than I'd like to admit...) but I simplified the system of cubic equations for the endpoints into this form.

```python
 pd = 1/(p2x-p1x)
 td = t-p1x
 ta = td*pd
 
 out(t) = (((p2s+p1s)/2 - (p2y-p1y)*pd)*ta*(t*2 + p1x - p2x*3) + (p2s-p1s)/2*(t + p1x - p2x*2))*ta + p2s*td + p1y
```

Isolated t to calculate coefficients in advance, providing a fast t evaluation:

```python
 pd = 1/(p2x-p1x)
 td = pd*(p2s-p1s)
 a = pd^2*(p2s + p1s - 2*pd*(p2y-p1y))
 
 b = (-a*3*(p2x + p1x) + td)/2
 c = p2x*(a*3*p1x + -td) + p2s
 d = p1x*(a/2*p1x*(-p2x*3 + p1x) + td*(p2x + -p1x/2) + -p2s) + p1y
 
 out(t) = t*(t^2*a + t*b + c) + d
```


I had the same question, was looking for a straightforward answer. Because my search query took me here and this is the closest I could find, I assume such solutions to this exact problem are not abundant, so I thought I'd share mine. I derived it because I need to emulate the same function outside of unity. Looking at the other three answers here, they seem to be correct but not simplified (trust me this is not an easy task) (Varaughe's leaves out the evaluation of the Bezier curve he made from the AnimationCurve, as it is a well-known thing).

I tested these against AnimationCurve for correctness with a Python script which can randomize all six parameters of the cubic function:

```python
 # isolated t
 def curve1(t,p1x,p2x,p1y,p2y,p1s,p2s):
     e = 1/(p2x-p1x)
     f = e*(p1s-p2s)
     a = (p1s + p2s + 2*e*(p1y-p2y))*e**2
     b = (a*3*(p2x+p1x) + f)/-2
     c = p2x*(a*3*p1x + f) + p2s
     d = p1x/2*(a*p1x*(p1x - p2x*3) + f*(p1x - p2x*2) - p2s*2) + p1y
     return t*(t**2*a + t*b + c) + d
 
 # short form
 def curve2(t,p1x,p2x,p1y,p2y,p1s,p2s):
     pd = 1/(p2x-p1x)
     td = t-p1x
     ta = td*pd
     return ((t*2 + p1x - p2x*3)*((p2s+p1s)/2 - (p2y-p1y)*pd)*ta + (t + p1x - p2x*2)*(p2s-p1s)/2)*ta + p2s*td + p1y
 
 # Paulius
 def curve_Paulius(t,p1x,p2x,p1y,p2y,p1s,p2s):
     xd = p2x-p1x
     t = (t-p1x)/xd
     return (2*t**3 - 3*t**2 + 1)*p1y + (t**3 - 2*t**2 + t)*p1s*xd + (t**3 - t**2)*p2s*xd + (-2*t**3 + 3*t**2)*p2y
 
 def test(curve, times, curve_parameters):
     out = []
     for i in times:
         out.append(curve(i, *curve_parameters))
     return out
 
 from random import random
 rand_args = []
 for i in range(4):
     rand_args.append(random()*100)
 for i in range(2):
     rand_args.append(random()*2)
 
 rand_times = []
 e,f = rand_args[0:2]
 points = 21
 # interval count is points - 1
 for i in range(points - 1):
     td = (f - e)/(points - 1)
     rand_times.append(random()*td + e + i*td)
```
