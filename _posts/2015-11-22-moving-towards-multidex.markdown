---
layout: post
title:  Moving Towards Multidex
author: Dmytro Voronkevych
date:   2015-11-22
categories: android
---
Android development is full of fun. Even more fun we have when trying to overcome
platform limitations.
One of such joyful and funny part is [`64K methods limit`](http://developer.android.com/tools/building/multidex.html).
If you follow the link, there is a solution to 64K limit, which seems simple
and sounds easy to go. Our practice showed some complications...

Preconditions
-------------
Badoo is heavily obfuscated. We do obfuscation with [`dexguard`](https://www.guardsquare.com/dexguard),
which is a proguard on steroids from the same author. Using this tool gives us many benefits, but
also put some limitation on toolset we can use and project setup we can have.

Stage1: Stay out from Multidex
------------------------------
Initially, when we faced 64K limit first time, there were no stable solution yet,
so we carefully analyzed all libraries we use and thrown away some of them.
It helped for some time, then we changed our code generator for data model objects
and got rid of some methods, like toString(), hasSomeField(), etc. It helped for
much longer.

Later, when we had to update google play services and compatibility library,
we had to setup dexguard to agressively optimize code in several phases.

This is what we put into dexguard confuguration file:
{% highlight proguard %}
-optimizations method/inlining/short, method/inlining/unique, method/inlining/tailrecursion, class/merging/vertical, !class/merging/horizontal, field/removal/writeonly, code/merging, code/removal/simple
-optimizations !class/merging/horizontal
-optimizationpasses 3
{% endhighlight %}

Build time become much slower and we split our build system into two branches:

* Build for contineous integration
* Build for developers

Maintaining them was two times more fun. Even more fun got our users, when app started
crashing in places where it never crashed before.

For example:
{% highlight java %}
protected synchronized Task awaitNextTask() {
    while (!Thread.currentThread().isInterrupted()) {
        Task result = findNextTask(mHighPriorityTasks);
        if (result != null)
            return result;
	result = findNextTask(mLowPriorityTasks);
        if (result != null)
            return result;
        try {
            wait(); // It is ok, when someone outside will interrupt this thread
        } catch (InterruptedException e) {
            // Retaining interrupted state
            Thread.currentThread().interrupt();
            return null;
        }
    }
    return null;
}
{% endhighlight %}
We got InterruptedException crash in our app. When we disassembled our obfuscated and optimized build,
we find out that try/catch block been optimized so much, that it just disapears.
No code - no problems (C) E. Stalin
> Note: this quote been slightly modified ;)

Quick fix was following:
{% highlight java %}
catch (InterruptedException e) {
    if (e.getCause() == null) { // Sorry for this line. We need to use exception, or Dexguard will remove catch block
        Thread.currentThread().interrupt();
        return null;
    }
    Thread.currentThread().interrupt();
    return null;
}
{% endhighlight %}

The reason why we still keep of from multidex was incompatibility between dexguard and multidex that days.

Stage2: Multidex is unevitable
------------------------------
Time flew fast, so as our product requirements. We had to integrate with more things and stuff. Write more code.
One day we found: there is nothing left we can remove from our code. And nothing more can be stripped off by dexguard for us.
This was the day when we won't able to release any more.

Luckily it happened some time after guardsquare announced support for multidex inside dexguard. We upgraded to a new version
of dexguard, played with setup a bit, faced several issues, nothing serios. I have seen a light in the end of a tunnel.

Until someone merged big change into release branch and new build started to crash with ClassNotFoundException. Funny part
about such crash that it happened when user pressed logout button, which happens usually when application is already started,
initialized, and used for some time. More fun included by the fact: crash occured only on devices with API < 21.

This is very important: on Android 21 we have an [`art runtime`](https://en.wikipedia.org/wiki/Android_Runtime),
which has embedded support for several dex files. So, classloader knows about all classes very early, before Application class is resolved and
instantiated. On [`dalvik`](https://en.wikipedia.org/wiki/Dalvik_(software)) we have an official, but a bit hacky solution to load
additional dex files in Application.onAttachContext() method. Till that moment, [`DexClassLoader`](http://developer.android.com/reference/dalvik/system/DexClassLoader.html) knows nothing about any classes located on secondary dex files.

Our particular issue was in this code snippet:
{% highlight java %}
public class BadooApplication extends MultiDexApplication {
    ...
    public static void logout() {
        ...
        Twitter.logout();
        ...
    }
    ...
}
{% endhighlight %}
ClassNotFoundException complainted about Twitter class. Obviously, Twitter class was put into second dex file, classloader failed to
resolve it on early stage, before second dex been loaded. This is interesting as Android suppose to have lazy class resolution
and suppose to resolve class only when it is used. I assume it does resolve classes on the link stage, but never complain about them,
until they used.

Maybe you remember old days when app crashed on android 1.6 on the code like this:
{% highlight java %}
if (Build.VERSION.SDK_INT > 9)
    return String.isEmpty();
else
    return String.length() == 0;
{% endhighlight %}

That days Dalvik classloader linked classes very early, when classes been loaded. Google changed that on later androids.

Conclusions
-----------
