
+++
date = "2016-02-16T10:36:05+01:00"
title = "khage"
type = "post"
+++

It has been a long time. Back for another article on Haxe. This time it relate to low level gpu programming with Khage (https://github.com/wighawag/khage).
The goal of khage is to simplify low level gpu programming so much that it become the primary way in which to render a game via code.

It might seem unecessary for those who do not want to deal with low level stuff and prefer to code with a scene graph or other higher level framework. 

In my experience (flash, cocos2d-x) these were never good idea. Most of the time, these scene graph trade too much performance and do not allow easy low level access when needed.
On top of that, they often end up as the substrate for the game's model and couple the logic and the renderring. This makes lots of task harder to do in the end.

The approach I prefer is to code the model of the game in anyway you want without any reference to rendering. Once the model is in place, you can then render it anyway you like. I usually execute opengl command there and I can organise the renderrign in layer (matching the gpu way). This is where khage comes in as it simplify gpu coding so much that it allow you to use it directly to render the model without losing type safety. Instead of dealing with shaders strings and attribute and uniform locations, you deal with normal type safe code.

Khage originated from a library I wrote a year ago for webgl games. It wrapped the webgl api and other browser js api into a potentially crossplatform api. The main interesting part from it was its use of a type safe wrapper arround glsl shaders.
It allowed me to write type safe concise code while dealing with low level gpu code.
It served me well and it was the first thing I wanted to port to my new destination : Kha (https://github.com/KTXSoftware/Kha).

I have heard of Kha before the WWX2015 but did not bother try another framework (which actually is not). I had already a far better idea : write my own :D

This was until I attended Robert's speech (http://www.silexlabs.org/kha/) and dived in Kha. This has now been almost 8 months with it and this is the best thing to happen to Haxe. It solves exactly what I was hoping to do one day with my js only library and a lot more. Its api works exactly the same across a variety of targets including flash, webgl, canvas, c++ (mobile, desktop...) and its architecture allow to add new platform quite easily. On top of that it contains a shader cross compiler (krafix : https://github.com/KTXSoftware/krafix) that can output to the different platform specific shading language from glsl.

My implementation of the type safe glsl wrapper for Kha is Khage (https://github.com/wighawag/khage). It is actually not finished but is already useful. What it does is it gather the name and type of  both attributes and uniforms at compile time. Then for each shader pair used it generates a class that contain a set of setter for the uniforms. This give you a type safe "program". In Kha's terminology this is actually a "pipeline" to match how gpu works internally and how the next generation of gpu api works (vulkan, metal...)
The pipeline wrapper also get a generated draw function that accept a specific buffer type (based on the attributes in the vertex shader).

These buffer types are generated via a @:genericBuild macro. To use a buffer that have position, texture coordinate and alpha you can create it this way :
```
buffer = new Buffer<{position:Vec3,tex:Vec2,alpha:Float}>();
``` 
The names here are important and such Buffer will only work with shader that have the exact same attributes (name and types). This might seem restrictive but allow type safe coding.

Khage also add some macro to make it simpler to use kha api. For example, kha render pipeline require you to wrap your rendering code with ```g.begin()``` and ```g.end()```. Khage add a macro that make the wrapping automatic.

Lubos has created a set of basic example for kha 3d usage (https://github.com/luboslenco/kha3d_examples) and I ported them over to use khage to show how it can simplify coding (https://github.com/wighawag/kha3d_examples). Compare https://github.com/wighawag/kha3d_examples/blob/master/tutorial04/Sources/Empty.hx with https://github.com/luboslenco/kha3d_examples/blob/master/tutorial04/Sources/Empty.hx
 
Currently the glsl parser is very basic and does not support comment or any other more advanced uniform (like struct and arrays). Krafix (the shader cross compiler) is able to output the uniform and attribute types and names. I plan to use it so it will works properly. I am waiting for some change in krafix to make this easier.

If you find this interesting, do not hesitate to drop a line and try khage (https://github.com/wighawag/khage)

Thanks for reading 