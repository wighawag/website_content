+++
date = "2011-06-05T16:26:20+01:00"
title = "Profiler Injection With MixingLoom as3commons bytecode and Flash Preloader"
type = "post"
slug = "Profiler-Injection-With-MixingLoom-as3commons-bytecode-and-Flash-Preloader"
+++

Recently I discovered **MixingLoom** through [James Ward's article](http://www.jamesward.com/2011/04/26/introducing-mixing-loom-runtime-actionscript-bytecode-modification/), it is a library to ease the injection of code in swf. One usage of it is [Aspect Oriented Programming](http://en.wikipedia.org/wiki/Aspect-oriented_programming) to separate logging, analytics... from the actual application code by injecting the extra code (logging, analytics...)  into the swf after compilation (thanks to [as3commons-bytecode library](http://www.as3commons.org/as3-commons-bytecode/index.html) ). This way the extra code does not appear anywhere in the application source code which stay focused on what it should do.

Here I ll show you how to inject profiler code to measure the speed of execution of particular functions. An xml file read at run time would be used to specify which function to profile (and to inject). It can be used for other purpose as well but since currently it does not handle arguments except for 1 string argument, it is quite limited.


### Preloader ###

Since I did not want to use flex at all, I could not use the example in James' article which use the MixingLoom flex preloaders.

Instead I used the built-in flash preloader, the one you specify through mm.config property (see [here](http://jpauclair.net/2010/02/17/one-swf-to-rule-them-all-the-almighty-preloadswf/) and [here](http://philippe.elsass.me/2010/09/as3-hacking-preloadswf-for-fun-and-profit/) ) it is a preloader preloaded by flash itself before anything is loaded. It allows us to execute code at the point where the main class is loaded.

This basically allow injection of code to any swf (whether it has been compiled by you or not). I had  some problem though making the profiler injection work when the swf is compiled in release mode. (see that later)

here is the preloader abstract class I use : 

<iframe width="100%" height="100%" src="http://script-iframe.appspot.com/script?url=http://gist-it.appspot.com/github/wighawag/mixingloom-profiler/raw/master/src/com/wighawag/preloader/AS3AbstractPreloader.as"><br /></iframe>


This abstract preloader need to be extended so that it is able to inject code into the application code. This can done by overriding  **applyModifications**  and then call **modificationsApplied** with the modified loaded bytes

**modificationsApplied** receive an event containing the modified application (the event can come directly from MixingLoom ModifiedByteLoader or PatcherApplierImpl (who reload the modified byte into flash).

The abstract preloader then add it to the stage.

In fact the abstract preloader will also remove everything from the stage so that it start clean and does not have two application running at the same time. 

Unfortunately, in many applications (since in most case, the main class is not supposed to be removed from stage) the main class has event listener registered with the stage which are not automatically removed when the main instance is removed from the stage. Because of that even if the main class is removed from stage the listener will keep the main class instance in memory and will be executed.

If you are the author of the application, you can simply unregister listener on REMOVED_FROM_STAGE event.

If you have no control over the code, then I do not know the solution, except maybe by analysing the swf bytecode to find listerner registration (a probably complex task) and unregister them.

The problem come from the built-in preloader : as soon as the main application is loaded it is instantiated and added to stage. It seems we do not have a hook before that happened. 

If anyone has a solution for it let me know.


Note : This abstract preloader can be extended for any purpose including non-mixingloom/injection one


### Setting up MixingLoom ###


Apart from having to re-hook the preloading part I also had to deal with the patchers and byte injection myself. Indeed James in his article use MixingLoom preloader which take care of applying the patch specified on the constructor through the flex specific code.

Thanks to MixingLoom it is just a matter of instantiating an IPatcherApplier and passing a vector of patchers

And thanks to the abstract preloader, I just needed to override **applyModifications** and call **modificationsApplied** when the patchers are applied (by setting the applier callback)

I created another abstract preloader extending the preceding one to deal specifically with patchers and applier:

<iframe width="100%" height="100%" src="http://script-iframe.appspot.com/script?url=http://gist-it.appspot.com/github/wighawag/mixingloom-profiler/raw/master/src/com/wighawag/preloader/AS3AbstractPatcherPreloader.as"><br /></iframe>

It just instantiate an PatcherApplierImp and override **applyModifications** to save the bytes that need to be modified and then expect its subclass to call **applyPatchers** with a list of patchers passed as argument

The **applyPatchers**  then add the list of patchers to the applier and set **modificationsApplied** to be the callback of the applier. The applier is then applied.

To extend the abstract class, the only neccessary bit is then to call **applyPatchers** with a list of patchers.

### Injection ###

Now that we have all the framework in place we can see an actual implementation of these abstract classes:

<iframe width="100%" height="100%" src="http://script-iframe.appspot.com/script?url=http://gist-it.appspot.com/github/wighawag/mixingloom-profiler/raw/master/src/MixingLoomAS3Preloader.as"><br /></iframe>

As you can see instead of dealing with xml loading in the patchers (as in Jame's example) I deal with them outside. This allow the patchers to focus on their actual work and also allow me to deal with loading failure directly in the preloader without requiring the patchers to tell us what happened.

on success the patchers are instantiated with the xml data and **applyPatchers** is called

on failure the patchers are not applied and it show an error message.

In both case I then add a console to the stage (in **xmlFailed** or **modificationApplied**). This [flash-console](http://code.google.com/p/flash-console/) is very useful to see what is going on and will allow us here to see the profiling results. It also allow to execute code dynamically through a command line interface. check it out.


If you noticed I imported com.wighawag.profiler.TimeProfiler but it is not used. I referenced it directly so that it is imported in the final swf.

This way I make sure I have this class compiled in when it is used by the injection.
Basically for each class method you want to inject you need it to be compiled in either in the preloader or in the swf target of the injection. In this project case I did not want the profiler to be part of the targeted swf instead I wanted the targeted swf to be clean of any profiling code or libraries.


Now let's discuss the actual patcher : **MethodCallWrapperPatcher**

The class is here:

<iframe width="100%" height="100%" src="http://script-iframe.appspot.com/script?url=http://gist-it.appspot.com/github/wighawag/mixingloom-profiler/raw/master/src/com/wighawag/injection/patcher/MethodCallWrapperPatcher.as"><br /></iframe>

its constructor expect xml data (not a url since it does not deal with loading). This xml data specify which method to inject and where

the apply function is the main method and looks quite similar to the SampleXMLPatcher of James except that it look for every tag (making it slower but make it potentially work for realeased swf)

Then for each class and for each of their instance it look whether their instance are specified as targets in the xml. 
If so, it inject the source classes' methods (so in this case it will inject the **com.wighawag.profiler.TimeProfiler** methods "start" and "finish" )


to do the actual injection I use **MethodCallWrapper** which can inject one method at the beginning of a method body and another method at the end (before every return)

Like this it can profile the time it takes to execute the function.


this methodCallWrapper takes two MethodCall which a list of argument **(currently only string arguments are accepted and only the first one will be taken in account)**

Here is the code:

<iframe width="100%" height="100%" src="http://script-iframe.appspot.com/script?url=http://gist-it.appspot.com/github/wighawag/mixingloom-profiler/raw/master/src/com/wighawag/injection/injector/MethodCallWrapper.as"><br /></iframe>

As you can see it follows the same template as the MixingLoom example: it does not change the byte before the pushscope opcode.

then it inject the first  MethodCall with the argument being the function being injected (so that the profile function knwo which function is beign measured)


Then it incrase the maxStack value so that even if the next method call is between a push opcode and a return opcode, the push opcode created by the method call will not exceed the maxStack (if it does, the flash player throw a Verify Error). 
It is probably possible instead of injecting the methodcall just before the return to detect where the last push opcode (used by the return opcode) is located and inject the methodcall just before it but it was easier to just increase the maxStack.


then for each return opcode, it inject the second MethodCall and return the position in the method body after injection so that we can chain injections

The MethodCall injector is as follow:

<iframe width="100%" height="100%" src="http://script-iframe.appspot.com/script?url=http://gist-it.appspot.com/github/wighawag/mixingloom-profiler/raw/master/src/com/wighawag/injection/injector/MethodCall.as"><br /></iframe>

It is similar to the MixingLoom method call injection except that it adds a string argument through a pushstring opcode

Again, after injecting it returns the position after the injection.


That's it. Normally it should work with application compiled in release mode but I get the error : **Cpool index 0 is out of range 47.** when the method call execute.
If I do not add the string as argument to the call, it execute fine.
I checked the as3commons-bytecode code and it correctly add the string to the constant pool so I do not know what is wrong. any ideas?

In debug everything is fine

the code is located at [github](https://github.com/wighawag/mixingloom-profiler) 
By the way, since the code shown in this page is dynamically taken from the github code, it is up to date


To show it works I created an example project at [github](https://github.com/wighawag/mixingloom-profiler-example)

<iframe width="100%" height="100%" src="/blog/content/mixingloomprofilerexample.swf"></iframe>

it has 4 button and 4 numeric stepper (from  [minimalcomps](https://github.com/wighawag/minimalcomps), actually a fork since there is issues with instance variables set at the class level , _maximum and _minimum in this case. I did not have time to investigate but it made me think that using mm.config preloader is not a very good idea even if it theorically allow you to profile any swf on the net.

the first button execute a recursive implementation of fibonacci for the value specified in the numeric stepper
By adding the path to the method in the xml, you ll see how long it takes to compute it in the console.

The second button execute an non-recursive implementation of fibonacci. It is also profiled assuming you use the provided injection xml.

the third and fourth button are not meant to be profiled since a wrapper function already profile it.
The time is shown next to the steppers. This allow  to see how the profiling injection affect the execution speed of the function injected.

you can download the compiled profiler [here](/blog/content/MixingloomPreloader.swf)

To see the profiling at work you need to edit your mm.config (see here) so that the flash preload your profiler

for example if the preloader has been downloaded in C:/Applications (you also need to add this path to the the trusted locations in [flash config](http://www.macromedia.com/support/documentation/en/flashplayer/help/settings_manager04.html))

you can use

    PreloadSwf=C:/Applications/MixingLoomPreloader.swf?xmlUrl=methodCallWrapperInjections.xml


the file **methodCallWrapperInjections.xml** is located in the bin fodler to which the url is relative:

<iframe width="100%" height="100%" src="http://script-iframe.appspot.com/script?url=http://gist-it.appspot.com/github/wighawag/mixingloom-profiler-example/raw/master/bin/methodCallWrapperInjections.xml"><br /></iframe>


it specify which function to profile including the fibonacci methods


You may have to download the [targeted swf](/blog/content/mixingloomprofilerexample.swf) to make it work and launch it directly (not sure if it works with browsers by default for example, I could not make it work with chrome)


And as you can see, the recursive fibonacci already slow in flash is even more slower with the profiling enabled. This is because the profiling function is called for each recursion and function call in flash is slow.

the code:

<iframe width="100%" height="100%" src="http://script-iframe.appspot.com/script?url=http://gist-it.appspot.com/github/wighawag/mixingloom-profiler-example/raw/master/src/Main.as"><br /></iframe>


The post have been longer (both in time to write and text length) than I thought and I hope it is clear enough. 

Thank you for reading! Do not hesitate to leave comments

