---
title: Creating isometric art in Blender
date: 2020-10-02
---

If, like me, you're passably familiar with Blender and not good at drawing you may want to use it
to create pre-rendered 2d artwork for isometric video game projects. I've found this works very
well but does require the scene to be configured correctly so that the rendered images are the right size for your game.

Most isometric 2d games aren't truly isometric, they're a slight variant of it to ensure a 2:1 gradient on 
horizontal lines so they can be represented nicely with pixels. Wikipedia has a separate page for isometric video game
graphics that has been very helpful for me at https://en.wikipedia.org/wiki/Isometric_video_game_graphics.

For this guide we'll render the blender default cube at the right scale for use in a video game with 64x32px isometric tiles.


## Preparing the cube

When creating a 3d object in blender it is easiest to work with blender units, which are the large grid lines
visible at the default zoom level. The default cube in blender is 2x2x2 blender units.

![Blender default scene](/blender-isometric/blender-cube.jpg)

We can leave the cube as 2x2x2 but it needs to be centered around the origin. The easiest way to do this is
to snap it to the grid.

## Positioning and rotating the camera

In video game isometric projection each shape appears rotated 45 degrees on the Z axis and 60 degrees on the X axis.
You can think of this as spinning a cube around (45 degrees on Z) and then tilting it downwards slightly so that you can see the top of it.

![Isometric projection](/blender-isometric/isometric-projection.png)

Importantly this isometric projection is *orthographic*. This means that unlike in a true-3d image there's no
sense of perspective. Items far away from you rendered in an isometric projection appear the same size as those that're closer.
You can see that the edges of the faces of the cube are all parallel to each other, while in a perspective projection they'd gradually converge.

To set these camera properties in Blender you need to make sure you're in object mode and then click on the camera in the 3d scene.
Orthographic projection can be set in the tab identified by a green camera icon and the rotation of the camera can be set in the 
tab identified by an orange square:

{{< video src="/blender-isometric/camera-settings.mp4" >}}

## Blender's Orthographic Scale

To achieve a correctly sized isometric image we need to make sure the cube completely fills the viewport when being rendered and
this means we need to set the correct orthographic scale. The orthographic scale can be thought of as being like a scale on a map.
A scale of 1 means that a 1x1 square, when viewed straight on, will completely fill a square viewport. A scale of 2 will mean that
the square will only fill half of the viewport, etc.

{{< video src="/blender-isometric/orthographic-scale.mp4" >}}

Unfortunately the isometric projection we're using makes setting the orthographic scale more difficult. Because the shapes we're
rendering are rotated by 45 degrees on the X axis we don't want a 1x1 square filling the viewport, rather we want the diagonal line
*across* the now rotated square to fill the viewport. Pythagoras' theorem tells us therefore that for a 1x1 plane the scale 
needs to be **1.4121** as the horizontal line across the square can be viewed as the hypotenuse of a right angle triangle:

![Isometric projection](/blender-isometric/horizontal-viewport-size.png)

Things get even more complicated with 3d shapes like our cube because we also need to take into account that the final shape is both
rotated 45 degrees on the X axis but also 60 degrees on the Z axis and this means the orthographic scales required are tricky to figure out.

## Setting the Orthographic Scale

I've written a small tool to calculate the correct orthographic scale for an isometric volume, hosted at https://tvc.io/blender.
Since the default cube is 2x2x2 tiles and our tile size is 32px we can plug these numbers into the tool, which I've embedded below:

{{< iframe src="https://tvc.io/blender" height="580" >}}

The result you should receive from the tool, given a 2x2x2 cube, is an orthographic scale of **1.57313** 
along with an image width of **32** and height of **36**.