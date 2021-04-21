---
title: "Dynamic Libraries"
date: 2020-09-05
categories: [Development,General]
tags: [libs,linker,rpath,runpath,ld]
---

## Anatomy

![image tooltip here]({{site.utl}}/assets/img/posts/PLT.png)

Libraries in Linux
* static
* shared
    * dynamic linking
	* dynamic loading

**Static libraries** contains functionality that is bound to a program statically at compile time.

**Shared libraries** are loaded when an application is loaded and binding occurs at run time. They can used in a couple of way: either <u>linked dynamically</u> at run time or <u>dynamically loaded</u> and used under program control. Linker takes the job of loading dynamically linked libraries while dynamically loaded libraries should be loaded manually.

## Linkers

* ld.so
* ld-linux-x86-64.so
* [...] depends on platform

Linker search the libraries in the following directories in the given order:
   + *RPATH* - a list of directories which is linked into the executable, supported on most systems (ignored if *RUNPATH* is present);
   + *LD_LIBRARY_PATH* - an environment variable which holds a list of directories;
   + *RUNPATH* - same as *RPATH*, but searched after *LD_LIBRARY_PATH*, supported on most recent systems (preferable);
   + */etc/ld.so.conf* - configuration file which lists additional library directories;
   + built-in directories - basically */lib* and */usr/lib*.

## Commands

Display the ELF headers or segments:
```bash
$ readelf -h <file> # display headers
$ readelf -l <file> # display segments
$ readelf -S <file> # display sections (each section is a part of segments)
```

There is section `.dynamic` which contains information about dynamic dependencies.

To show dependency list of dynamic linking executable:

```bash
$ ldd <file>
# or
$ readelf -d main | grep NEEDED
# or
$ /lib64/ld-linux-x86-64.so.2 --list <file> # (depends on used linker)
```
List symbols in file:
```bash
$ nm -CnD <file>
```

## Configuration

Configuration file: **/etc/ld.so.conf**
Configuration Commands : **ldconfig**

The main purposes of configuration command are processing configuration file and creating cache to speed up loading of applications.

There are two available ways to specify additional place of libraries:
* Change configuration file;
* Use LD_LIBRARY_PATH environment variable;
* Change rpath or runpath.

### Change configration file

Update _/etc/ld.so.conf_ configuration file and after that you have to update cache. To do this you need to run `ldconfig` command by root.
To check cache entries you can run `ldconfig -p` command.

### Use LD_LIBRARY_PATH environment variable

To run particular executable and specify additional place of libraries use next command line:
```bash
$ LD_LIBRARY_PATH=... <path-to-executable>
```

To use this behavior for each user session you can export this environment variable in running script (e.g. `$HOME/.bashrc`).

### Change rpath or runpath

Both rpath and runpath are list of paths to search for library dependencies. The only difference between them is the order they are searched in. Specifically, their relation to `LD_LIBRARY_PATH` - `rpath` is searched in before while `runpath` is searched in after. The meaning of this is that `rpath` cannot be changed dynamically with environment variables while `runpath` can.

The best way to change value of `rpath` is to specify it while compiling of executable. Refer to a good explanation how to do this by cmake - [RPATH - handling](https://gitlab.kitware.com/cmake/community/-/wikis/doc/cmake/RPATH-handling).

Another way to change value of `rpath` is using `patchelf`:
```bash
$ patchelf --set-rpath /path/to/your/libdir:/path2/to/your/libdir/:$ORIGIN <executable>
```
The $ORIGIN variable could be used to represent for the current directory where you put your executable.

## Debug

To dump debug information of dynamic linker which can be used to resolve loading problems use `LD_DEBUG` environment variable. To see available options run any program with the variable set to `help`:
```
LD_DEBUG=help cat
Valid options for the LD_DEBUG environment variable are:
...
```
Example of using this environment variable to find out where dependencies are searched:
```
LD_DEBUG=libs ldd cat
...
```
The output of the command is also very useful for tracking performance problems for excessive dynamic loading.

## Create

### Static Library

To compile application and link with two static libraries follow next commands:
```bash
$ gcc -c A.c
$ gcc -c B.c
$ ar cr libA.a A.o
$ ar cr libB.a B.o
$ gcc main.c -lA -lB -L.
```

Above commands do:
* Create object file of library A;
* Create object file of library B;
* Create static libraries from these library object file;
* Create executable file and link with two external static library `A` and `B`.

### Dynamic Library

To compile application and link with two dynamic libraries follow next commands:
```bash
$ gcc -c A.c -fPIC
$ gcc -c B.c -fPIC
$ gcc -shared A.o -o libA.so
$ gcc -shared B.o -o libB.so
$ gcc main.c -L. -lA -lB -Wl,-rpath,.
```

Above commands do:
* Create object file of library A;
* Create object file of library B;
* Create dynamic libraries from these library object file;
* Create executable file and link with two external dynamic library `A` and `B`.

**Note:** One of the most significant thing is option PIC (position independent code). This option instructs to generate valid address of the extern methods when calls take place.

## Replacement

To make some method replace and redirect to the specific implementation inside library use `LD_PRELOAD` environment variable. This variable tells the linker to populate the PLT with symbols from this library and skip already loaded symbols from other libraries. Such behaviour is the same when symbols from standard library considered. 

```bash
$ LD_PRELOAD=/path/to/some.so main.x
```