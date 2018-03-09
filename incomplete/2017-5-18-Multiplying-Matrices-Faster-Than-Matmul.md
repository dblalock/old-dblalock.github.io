
---
layout: post
comments: true
title:  "Multiplying Matrices Faster Than a Matrix Multiply"
excerpt: "Machine learning meets SIMD instructions"
date:   2017-5-21 15:20:00
---

If there's any operation fundamental to scientific computing, it's the matrix multiply. This is probably the most heavily optimized routine in existence, and research on practical algorithms to accelerate it has more or less come to an end.

Or has it?

It turns out that matrix multiplies can be made much faster if we're willing to accept a little bit of error in the results. This is known as an approximate matrix multiply, and is still an active subject of research.

There are a few algorithms for this worth highlighting here

