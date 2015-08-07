+++
date = "2012-12-31T16:26:02+01:00"
title = "NME Stage3D"
type = "post"
aliases = [
    "/blog/2012/12/NME-Stage3D-support"
]
+++

Hi everybody, long time I did not write a post,

During that time I learnt a wonderful language, [Haxe](haxe.org). I have still lot to learn thought. I don't want to go in the details of why this language (or more accurately its ecosystem) is awesome but as an Actionscript developer for the last 3 years, there is no question to switch back. Haxe offer everything that actionscript has to offer (compiling to swf including air) + many extra feature

The ones I like the most are:

- Generics,
- Typed function
- and Macros (which I still have to practice more)

I also like that the compiler is very fast (compared to mxmlc) and it handle Type inference, no need to repeat yourself like in actionscript


But the main advantage is that Haxe can compile to other languages. The current stable ones being swf, javascript, php, c++ and neko (a virtual machine that I did not use much except for writing tools and for using Macros (as they run on this virtual machine).

This not only means that you can get your code working on android, ios, blackberry, linux, mac os x, windows, webos but also that you can share code between client and server

If I find some time I would like to make a Python target myself but as of now this is unlikely to happen


The other great thing about Haxe is its community, they are doing a great job with Haxe itself (haxe 3 in the making) and with great libraries ([NME](http://www.nme.io/), [Flambe](https://github.com/aduros/flambe), [Promhx](https://github.com/jdonaldson/promhx),.,) and quick to reply to questions


Let's get back on the topic. I am writing this post to share the last work I did regarding Haxe and maybe this will contribute to the community..

I am currently working on a game engine which I plan to use for mobile game as well as web based game (thanks to Haxe of course :)

I have a renderer agnostic api and until the last few days I was using:

- Flambe 1.2 (only its rendering part) to render stuff on Flash (using stage3d and bitmap renderer as a fallback)
- NME Tilesheet API for cpp target (testing only linux and android)

Then I started to implement a tile map and as soon as I started to add many tiles, my old android phone (an HTC magic upgraded to androdi 2.3) could not display anything,

The reason was not Tilesheet performance (using GPU) per se but the fact that in order to update the position of the tiles while scrolling, the CPU (not the GPU) could not handle the many tile. Of course there were lots of optimization possible as I was using a quite high level API but this problem made me think that it would be nice if I could use the GPU myself directly so that I can save my vertex buffer at the beginning and at each frame I would just need to update the view matrix.
Actually maybe there was a nice way to achieve the same with tilesheet API ?

Anyway, just in time came NME 3.5.1 with support for OpenGLView

I jumped on it and learn OpenGL on the way to realize that it should not be too difficult to implement the Stage3D API for NME

So here it is [https://github.com/wighawag/NME/tree/stage3d](https://github.com/wighawag/NME/tree/stage3d)

It is still not fully working as flash Stage3D is. here is a partial list of the issues:

- there is no support for multiple textures yet
- (Index/Vertex)Buffer can only be uploaded from ByteArray as of now (it might be straigthforward to make it work with Vector though)
- for Context3D here are the following issues:
- Context3D.setCulling is not tested
- Context3D.setDepthTest not woking  (Currentlty the constants for the Depth test are not set in the opengl API provided in NME)
- Context3D.configureBackBuffer does not take in consideration antiAlias and enableDepthAndStencil arguments
- Context3D.createCubeTexture not implemented
- Context3D.createTexture does not consider Format, optimizeForRenderToTexture and streamingLevels arguments
- Context3D.drawToBitmapData not implemented
- Context3D.setProgramConstantsFromByteArray not implemented
- Context3D.setProgramConstantsFromVector not implemented
- Context3D.setRenderToBackBuffer not implemented
- Context3D.setRenderToTexture not implemented
- Context3D.setScissorRectangle not implemented
- Context3D.setStencilActions not implemented
- Context3D.setStencilReferenceValue not implemented
- probably many thing I forgot
- since the openGL api used in NME does not allow direct call to render, you need to set a render function. This is achieved by passing a function to Contex3D.setRenderMethod (which does not exist in flash). To be as close as possible to flash Stage3D I should probably allow to render in Event.ENTER_FRAME or any timer. But this would require to cache the calls made to context3D and replay them when the openGL API call the render function.

Anyway the main drawback currently is that since Stage3D and opengl use different shader language you still need to code your shader in two languages. Also regarding GLSL, since I wanted to use Stage3D api which does not support arbitrary name for attributes and uniforms, you have to use the same convention as AGAL (vc0, vc1 for vertex uniforms, va0, va1.. for vertex attribute, ...etc...) as attribute and uniform name in GLSL

The best solution here would be to extend [HXSL](http://haxe.org/manual/hxsl) and use it to have only one language for both Stage3D and GLSL but I did not look into that. I know there were some work on a GLSL target for HXSL but I don't know what is the state of this as of now, Any info ?

I will porbably look into it when I ll need more complex shader stuff.


I plan to continue working on this Stage3D port and I would be very glad if it could be included in NME trunk so my work do not get lost as a patch file.  I ll probably do a pull request even if it s still not perfect)


I posted a test here  [https://github.com/wighawag/NMEStage3DTest](https://github.com/wighawag/NMEStage3DTest)

<iframe width="500" height="500" src="/blog/content/NMEStage3DTest.html"></iframe>

It should work in Flash without my new version of NME
And the patch provided [here](https://raw.github.com/wighawag/NMEStage3DTest/master/stage3d.patch) should make it work in cpp (tested only on Linux 64 bit by the way)

You ban also clone my NME fork if you prefer and checkout the "stage3d" branch


Let me know what you think and if some of you already started some similar work, let's cooperate.


All the best for the coming new year!


We survived the end of the world so let's celebrate apropriately :)
