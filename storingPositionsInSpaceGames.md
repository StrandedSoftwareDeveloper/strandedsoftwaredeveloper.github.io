# Storing position in space games

*Note: This article assumes basic knowledge of the concept of storing positions with numbers, and*
*the concept of representing the same position in different reference frames/coordinate systems.*

Most 3D games store the positions of everything with something called "floating point numbers", which can store a very
wide range of values. They do this by being very precise when the number is very small, and getting less and less
precise as the number gets bigger. While this usually works fine, in some cases, particularly space games, this can
be a bit of an issue.[¹](#footnote-1) This article covers the basics of what this problem is, and two different solutions to it.

# The problem
As mentioned, when floating point numbers ("floats", though that often refers specifically to 32 bit ones) are small,
they're quite precise, but as they get bigger (as they often do in space games), they get very coarse. At a distance
of 2 meters from the origin, a single precision (32 bit) float has a precision of about 0.2 microns,[²](#footnote-2) whereas at a
distance of 65,536 meters, it has a precision of only 1 centimeter, and at 8,388,608 meters (8,388 km, only 1/5th
of the way to geostationary orbit) it has a precision of a whole meter.

Clearly these won't do.

Double precision (64 bit) floats ("doubles") on the other hand, have a precision of 0.4 *femtometers* at 2 meters
from the origin. Wolfram Alpha tells me that's smaller than an electron,[³](#footnote-3) At 8,388,608 meters the precision is
2 nanometers. They only reach a precision of 1 meter at 4,503,599,627,370,496 meters (*Almost half a light year!*)
from the origin. At the edge of the Kuiper belt, a double has a precision of 1 millimeter, which I reckon is about
the edge of what's acceptable for rendering and physics.

So, can't we just use doubles for everything?
Not quite.

Technically we could, but GPUs are generally very slow at processing doubles, if they're supported at all.
Like, up to 32x slower[⁴](#footnote-4) on some cards.
So what are we to do?


# KSP (1 & 2)'s solution
Unity's physics doesn't support doubles, so they're *really* not an option for KSP. Instead, KSP uses a system that
I thought was called "Floating Origin", but I found some things[⁵](#footnote-5) suggesting that it's instead called a
"threshold based shifting method". Either way, here's how KSP does it[⁶](#footnote-6): In KSP, the origin moves ("floats")
to always be near the active vessel. When near the ground (under 150 meters above the terrain), the origin will lock in
place, teleporting towards the active vessel whenever it gets too far away. You can actually hear this in the game as
brief cuts in the sound effects of the engines. When above 150 meters, the origin will snap to the active vessel,
matching it's velocity so that the center of the active vessel is always at (0, 0, 0) with velocity (0, 0, 0),
with everything else moving around it instead. So the basic idea is to move the origin so that anything far
enough away to suffer precision issues is:
* Out of physics range so it's not a problem for the physics, and
* Too far away for the player to see rendering artifacts

However, this system adds a lot of complexity and, if not done carefully, can introduce quite a few bugs.


# The "premultiply the matrices" solution
This is a solution I thought of, and I suspect just *might* be the solution RocketWerkz is using for they're in-(early)-development
game "Kitten Space Agency" ("KSA", working title). *(Edit: We have confirmation from RocketWerkz that this is indeed how it works in KSA)*
The physics/logic part of the idea is:
* Store all positions, velocities, etc. as doubles on the CPU
* Use a physics engine that supports doubles

For the rendering part, we first need some background:  
In the vast majority of game rendering systems (including the one KSA is using), the rendering process looks something like this:
* Start with the positions of the triangles relative to the 3d model
* Multiply the positions by a matrix called the "model" or "world" matrix to move them to be relative to the position of the 3d model in the world (or solar system or whatever)
* Multiply that by another matrix called the "view" matrix to move them to be relative to the position of the camera
* Multiply by the "projection" matrix (and then divide by the `w` component) to squish far away things and stretch close things to achieve perspective (so closer things take up more of the screen as they should)
* Throw away[⁷](#footnote-7) the z coordinate
* You now have 2d coordinates of a bunch of triangles
* Render a (2d) triangle at each of those positions on the screen

It's ok if you don't understand some of the math there, the important parts are:
* The final positions of these triangles is (partially) determined by these matrices, and
* Matrices can be multiplied with each other to get a matrix that, when multiplied with
    a position, has the same effect as multiplying it with each of the matrices it's composed of.

Now, as you may be able to see, the camera, in a sense, doesn't really exist in the rendering system.
It's just a bunch of triangles at different positions. All the "camera" does is tell the rendering
system how to move the triangles so that they end up at the right spot on the screen.

In a hypothetical game that has the camera and some object really far away from the origin, the model/world
matrix and the view matrix are the only two that have big numbers in them. The important parts however are
exactly *what* numbers are in those matrices and the positions of the triangles after being multiplied by
those two matrices.

In this process, the final positions of those triangles are *near the camera*, and thus close to the origin
of that coordinate system (the one where the camera is in the center). Since we know that the triangles
start out (relatively) close to the center of the 3d model, and are far far away from the origin in
"world space" (relative to the main origin, the center of the sun or whatever), we know that the
model/world matrix needs to have some pretty big numbers to get them out there, and the
view matrix must have some big numbers to get them back.

So here's the core of the idea:

If we effectively move the triangles really far away and then immedietly move them back, why don't
we just multiply the model/world matrix and the view matrix together in double precision on the CPU
so that the big "move far away" numbers and the big "move back near the center" numbers cancel out,
leaving us with relativly small numbers? Then we just convert the result down to single precision
and send it off to the GPU.

The only problem then is that most shaders use the position in world space for lighting calculations.
But as long as you move the light sources too, all the lighting math should work just as well in
"view space" (relative to the camera).

It's worth mentioning that at the time of writing, I have not yet tested this method, though
I don't have any reason to believe it wouldn't work.


## Performance
I decided to see what the performance overhead would be to premultiply the view matrix with each model/world
matrix on the CPU.

With a simple test program multiplying random 4x4 matrices together and counting the cycles for each multiplication,
my i5-4200m gets about 40 cycles per multiplication with several other programs open. At 2.5 GHz, that equates to
a theoretical average of 62,500,000 multiplications per second, or about a million objects at over 60 fps. Though
in reality, the program only achieves 10,000,000 in 7.19 seconds for a total of 1,390,820 multiplications per
second including generating the matrices.

Since a reasonably designed game should have significantly less than 1,000,000 objects, I imagine the performance
isn't much of a concern.


# Feedback
This is the first article I've ever posted, so (constructive) feedback/critisism is appreciated.
Please send feedback to the [issue tracker](https://github.com/StrandedSoftwareDeveloper/strandedsoftwaredeveloper.github.io/issues) for this website.


# Footnotes
<a id="footnote-1">¹</a> [https://www.youtube.com/watch?v=wGhBjMcY2YQ](https://www.youtube.com/watch?v=wGhBjMcY2YQ)  

<a id="footnote-2">²</a> You can calculate this and all the other precision values [here](https://evanw.github.io/float-toy/) by seeing how much change flipping the rightmost bit causes when the number is set to the overall size you want to check  

<a id="footnote-3">³</a> [https://www.wolframalpha.com/input?i=0.4+femtometers](https://www.wolframalpha.com/input?i=0.4+femtometers)

<a id="footnote-4">⁴</a> From [https://en.wikipedia.org/wiki/Maxwell_(microarchitecture)#Performance](https://en.wikipedia.org/wiki/Maxwell_(microarchitecture)#Performance) : "The theoretical double-precision processing power of a Maxwell GPU is 1/32 of the single precision performance"  

<a id="footnote-5">⁵</a> This reddit comment, apparently from the inventor of Floating Origin: [https://www.reddit.com/r/KerbalSpaceProgram/comments/11hcnyt/comment/jazcq9f/](https://www.reddit.com/r/KerbalSpaceProgram/comments/11hcnyt/comment/jazcq9f/)  

<a id="footnote-6">⁶</a> Described in detail in this part of this talk by KSP developer Felipe Falanghe (AKA "HarvesteR"): [https://www.youtube.com/watch?v=mXTxQko-JH0&t=377s](https://www.youtube.com/watch?v=mXTxQko-JH0&t=377s)  

<a id="footnote-7">⁷</a> We usually keep the z coordinate around to make sure closer things render in front of further
  things, but I thought the whole projection thing might be a bit clearer if I specified that
  we don't use the z coordinate for this part.
