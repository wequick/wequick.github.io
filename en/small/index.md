---
layout: project
permalink: /small/
title: Small
subtitle: A <strong>small</strong> framework to split app into <strong>small</strong> parts
github: https://github.com/wequick/small
icon: /assets/images/small-icon-72.png
javadoc: docs/javadoc/0.3.0/
---

### Overview

Small is a framework that help you to split your App into small patches of Plug-Ins.

* **Perfect built-in**
  - All plugins are support to build in host application.
* **Highly transparent**
  - The plugin codings (code, layout, etc.) are as same as a single application.
  - Support plugin debuging just like a completion application.
* **Ultimate slicing**
  - Splits out any shared codes and resources from plugins.
* **Seamless connection**
  - The host, native app bundle, native web bundle, online web page and any custom bundle can launch and pass parameters to each other with a simple uri.
* **Cross platforms**
  - Until now, we support android, iOS and html5 plugins. In addition, they can communicate with each other by an uniform javascript interface.

Cause the Plug-Ins development is not an official solution, we should take some time to configure the project.

Small supports Android 4.0.3 and above. For iOS, the minimum requirement is 7.0.

### Configuration

#### Android

##### Add Small library in your root `build.gradle`. (The latest versions are on [here](https://bintray.com/galenlin/maven))
<pre class="prettyprint">
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:1.3.0'
        // Small library to compile plug-ins
        classpath 'net.wequick.tools.build:gradle-small:0.1.2'
    }
}
apply plugin: 'net.wequick.small' // Small library to launch plug-ins
small {
    aarVersion = '0.3.0'
}
</pre>

##### Create module

![New small module][anim-new-md]

The module created should follow following naming conventions:

* The module name should be `"[module type].*"`
* The module package name should be `"*.[module type].*"`

The module type supports:

* **app** - the application plugin
* **lib** - the library plugin which can be included by the **app**
* **web** - the native web plugin
* **any other** - your customize assets plugin

##### Setup

Create `bundle.json` file to your host module's asset folder.
<pre class="prettyprint">
{
  "version": "1.0.0",
  "bundles": [
    {
      "uri": "main",
      "pkg": "com.example.mysmall.app.main"
    }
  ]
}
</pre>
Place the setup code in your host module's launcher activity.
<pre class="prettyprint">
Small.setUp(this, new net.wequick.small.Bundle.OnLoadListener() {
    @Override
    public void onLoad() {
        Small.openUri("main", LaunchActivity.this);
    }
});
</pre>

#### iOS

_Coming Soon._

### Building

#### Android

##### Build libraries

    > [./]gradlew buildLib -q
    
  ![Build libraries][anim-bL]
    
##### Build bundles

    > [./]gradlew buildBundle -q
    
  ![Build bundles][anim-bB]

[anim-new-md]: http://code.wequick.net/assets/anims/small-new-module.gif
[anim-bL]: http://code.wequick.net/anims/small/android-build-lib.gif
[anim-bB]: http://code.wequick.net/anims/small-android-build-bundle.gif