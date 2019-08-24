---
layout: post
title: Customising the dist source archive with CMake and CPack
---


![cmake]({{ "/" | relative_url }}public/img/2019-08-22-cmake.svg)

Autotools provides a `make dist` target that packages up all required source files into a standalone, distributable tarball. CMake provides similar functionality through CPack, although on the surface, its configurability is quite limited. With CPack documentation lacking, this post aims to address how that dist tarball can be customised - specifically in adding generated files to it. 

<!--more-->

## CPack mini-primer

CPack is a tool bundled with CMake that facilitates creating source and binary redistributable packages. While CPack can be used in a standalone fashion, in the context of CMake, it is usually invoked by setting up a bunch of variables, then including the `CPack` module:


```cmake
set(CPACK_PACKAGE_NAME name)
# ... set up more variables ...
include(CPack)
```

This creates two targets, `package` and `package_source`, in addition to creating two config files in the build folder, `CPackConfig.cmake` and `CPackSourceConfig.cmake` for the two targets respectively. The contents of those are largely determined by what gets set before including the `CPack` module.

I will be focusing only on the `package_source` target, and not the binary packaging functionality. With this in mind, it's possible to alias the `package_source` to the more familiar `dist` target:

```cmake
add_custom_target(dist
    COMMAND "${CMAKE_COMMAND}"
      --build "${CMAKE_BINARY_DIR}"
      --target package_source
    DEPENDS <dependencies>
    VERBATIM
    USES_TERMINAL
  )
```

In addition to aliasing the target, this also allows us to add dependencies to it, such as downloading or generating files that you wish to include into the bundle.

## How CPack knows what to bundle

What gets bundled in the source package is largely controlled by the `CPACK_SOURCE_INSTALLED_DIRECTORIES` variable. This is a pairwise defined list, where each pair defines the source to copy from, and the target destination in the package. When invoked from CMake, it defaults to copying the entire source folder into the root of the dist package. For example, if your source was located at `/home/user/source`, then it would be defined as:

```cmake
set(CPACK_SOURCE_INSTALLED_DIRECTORIES "/home/user/source;/")
```

Note that CPack blindly copies anything in that folder, so you should generate dist packages from a clean checkout. You can also set `CPACK_SOURCE_IGNORE_FILES`, which is a list of regexes, of which to ignore when searching for files to bundle.

## Bundling extra files, first take

To bundle extra files, [this post](https://cmake.org/pipermail/cmake/2011-January/041870.html) suggests extending `CPACK_SOURCE_INSTALLED_DIRECTORIES` with something like:

```cmake
set(CPACK_SOURCE_INSTALLED_DIRECTORIES
  "${CMAKE_SOURCE_DIR};/;${CMAKE_BINARY_DIR}/extras;/extras")
```

However, a common setup is to place the build/binary folder within the source folder. If you've set up `CPACK_SOURCE_IGNORE_FILES` properly, and your binary folder is within the source directory, then that ignore directive takes precedence, and the above will have no effect.

Of course, if you place your build folder completely outside of the source folder, then this should work.

## Bundling extra files, take two

Luckily, it is possible to instead invoke a script that runs when the package is made. This allows you to do almost anything to the bundle, but at the very least, it allows you to have the build folder *within* the source folder, *and* have generated files in that build folder be added to the package.

This is controlled by the `CPACK_INSTALL_SCRIPT` variable. It should point to the path of a CMake script to be invoked. Note that it is invoked right before anything is actually copied into the staging folder for the package.

Also of note is that `CMAKE_CURRENT_BINARY_DIR` from within the install script is set to the path of the staging folder for the package's contents. Other variables normally accessible from CMake are not passed into the install script. To work around this, we can just generate the install script, substituting in any required variables.

With this in mind, this is a minimal example:

`my_install_script.cmake.in`:
```cmake
if(CPACK_SOURCE_INSTALLED_DIRECTORIES)
  file(
    INSTALL "@CMAKE_BINARY_DIR@/extras"
    DESTINATION "${CMAKE_CURRENT_BINARY_DIR}"
  )
endif()
```

From where `CPack` is included:

```cmake
configure_file(my_install_script.cmake.in my_install_script.cmake)
...
set(CPACK_INSTALL_SCRIPT "${CMAKE_CURRENT_BINARY_DIR}/my_install_script.cmake")
include(CPack)
```

In the install script, I check if `CPACK_SOURCE_INSTALLED_DIRECTORIES` is set - since this install script is also run in other modes (supposing you've set up CPack to make binary packages), this is my method to know that CPack is currently bundling a source package.

This is the approach I took to bundle extra test fonts into the dist bundle for FontForge - you can see this in action [here](https://github.com/fontforge/fontforge/blob/master/cmake/CPackSetup.cmake).

For more complex use cases, it's also worth mentioning that you can set the `CPACK_PROPERTIES_FILE` variable to yet another CMake source file. This is included at the end of the `CPackSourceConfig.cmake`. From there, it is possible to override variables set earlier in that config file, and to set up some additional state before your install script is run. For example, in [this commit](https://github.com/fontforge/fontforge/commit/a8d6b79e1081d6804d9c7345e39edb533afa22c7), I was experimenting using it to determine some additional versioning information for setting up Debian packaging metadata.
