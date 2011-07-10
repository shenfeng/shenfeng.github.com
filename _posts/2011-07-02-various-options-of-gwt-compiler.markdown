---
author: feng
date: '2011-07-02 14:13:46'
layout: post
status: publish
title: Various options of GWT compiler
categories:
- java
- technology
tags:
- gwt
---

**build.xml:**
{% highlight xml %}
   <property name="gwt.args"  value="-compileReport-localWorkers 4
                              -XdisableClassMetadata -XdisableCastChecking" />
{% endhighlight %}

**gwt.xml**
{% highlight xml %}
   <set-property name="user.agent" value="safari,gecko_8"/>
   <set-property name="compiler.stackMode" value="strip"/>
   <!--removes client-side stack trace info (can reduce size up to 15%)-->
   <set-configuration-property name="compiler.enum.obfuscate.names"
                               value="true" />
   <!-- only use if you're not using enums as String values -->
   <set-configuration-property name="CssResource.obfuscationPrefix"
                               value="empty" />
{% endhighlight %}

