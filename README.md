This repository can be used to illustrate a compile problem when using Android NDK 26 or later in Visual Studio.

Environment: VS 2022 17.11.0 Preview 2.1 on Windows 10 (19045.4529).

Setup:
- Create a simple Android static library (New Project -> Static Library (Android)).  I updated the one here to use the android-33 API
- (Configuration Properties -> General -> Target API Level in project properties).
- Via Tools -> Android SDK Manager, install Android SDK Platform 33 and NDK 26.3.11579264 (Tools -> Other -> NDK (Side by side)
  26.3.11579264).  The former will end up in "C:\Program Files (x86)\Android\android-sdk" and the latter in
  "C\Program Files (x86)\Android\android-sdk\ndk\26.3.11579264" (or wherever your 'Program Files (x86)' folder is).
- In VS under Tools -> Options -> Cross Platform -> C++ -> Android, set Android SDK and Android NDK to the paths mentioned above.

Problems:
1. Going from NDK 25 to 26, 'lib64' in the path to clang was consolidated into the existing 'lib' folder.  This is a problem because
   if you go to <Edit>... under Configuration Properties -> VC++ Diretories -> Include Directories in the project properties, you'll see
   that "$(LLVMToolchainPrebuiltRoot)\lib64\clang\$(LLVMVersion)\include" is an inherited value.  We have to uncheck 'Inherit from
   parent or project defaults', and manually add the new directory:  "$(LLVMToolchainPrebuiltRoot)\lib\clang\$(LLVMVersion)\include".
   This isn't really a problem, just more of an out-of-date inconvenience.

2. Going from NDK 25 to 26, "[NDK Root]\sources\cxx-stl\llvm-libc++" and "[NDK Root]\sources\cxx-stl\llvm-libc++abi" were removed.
   These folders (which contain the C++ headers) are added as the first header search paths by default via $(StlIncludeDirectories),
   assuming you have 'Inherit from parent or project defaults' checked.  Immediately after these two folders get added,
   "$(SysRoot)\usr\include" (C headers) and "$(SysRoot)\usr\include\c++\v1" (C++ headers) get added automatically (no matter what you
   do - you can't change this or their ordering). In this instance, the order is correct: C++ headers will be found first, and C headers
   second, thanks to "llvm-libc++" existing.

   However, in NDK 26 where "sources\cxx-stl\llvm-libc++" and "sources\cxx-stl\llvm-libc++abi" are gone, the C headers are found
   first as "$(SysRoot)\usr\include" comes before "$(SysRoot)\usr\include\c++\v1", which is backwards.  And if you try and include
   a C++ header (say, #include <cstdlib>), you'll get this annoying-but-really-helpful error message:

   error : <cstdlib> tried including <stdlib.h> but didn't find libc++'s <stdlib.h> header.  This usually means that your header
   search paths are not configured properly. The header search paths should contain the C++ Standard Library headers before
   any C Standard Library, and you are probably using compiler flags that make that not be the case.

Workaround:

I worked around this (hopefully temporarily) by renaming the actual "$(SysRoot)\usr\include" folder of my NDK 26.3.11579264 installation
to "$(SysRoot)\usr\inc".  This means that the default header search path addition will fail because the directory doesn't exist.
I can subsequently manually specify "$(SysRoot)\usr\inc" as a header search path AFTER I specify "$(SysRoot)\usr\inc\c++\v1"
as one (along with "$(Sysroot)\usr\include\$(AndroidHeaderTriple)", to be complete).  Problem "solved"; everything builds and runs fine.

Notes:

I added the -v switch to the clang compiler command via Configuration Properties -> C/C++ -> Command Line -> Additional Options
in the project properties dialog to see the list of include paths in the Output window in the ordered that they'll be searched
when you compile.  That really helped.
