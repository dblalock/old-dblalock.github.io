---
layout: post
comments: true
title:  "Hierarchical builds via Scons"
excerpt: "This is a test. This is only a test"
date:   2014-10-5 22:54:00
---

If you want cross-platform C/C++ builds, there are basically [only two real options](http://stackoverflow.com/a/18291580):
[CMake](http://www.cmake.org) and [Scons](http://www.scons.org). CMake is faster and more widely used, but appears to be kind of a pain and requires learning a special scripting language. In constrast, Scons is a bit less polished, but is simple and allows one to write build files in pure Python.

I recently had to get a ~10kloc project building on multiple platforms and so, knowing Python and not needing anything fancy, I decided to try Scons.

Unfortunately, while the documentation makes clear how to do *extremely* simple builds wherein everything is the same directory and/or has no dependencies to other directories, it's not clear how to do a hierarchical build with a realistic project structure.

To save anyone else out there the pain of figuring out how to do this, here's an [example hierarchical build via Scons](https://github.com/dblalock/scons-example). The basic idea is that there's a root "Sconstruct" file that kicks off the whole build by calling "sconscript" files in the roots of the source and test directories.

This system will work regardless of your source and test directory structures, since it recursively adds all source directories to the build path. You could also recursively call custom "sconscript" files within each directory if you wanted to ensure that there weren't bizarre dependencies between modules, but for a small-ish project, the include-everything-ever solution is convenient.
