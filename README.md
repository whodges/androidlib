This repository can be used to illustrate a compile problem when using Android NDK 26 or later in Visual Studio.

Environment: VS 2022 17.11.0 Preview 2.1 on Windows 10 (19045.4529).

Setup:
- Create a simple Android static library (New Project -> Static Library (Android)).  I updated the one here to use the android-33 API (Configuration Properties -> General -> Target API Level in project properties).
- Via Tools -> Android SDK Manager, install Android SDK Platform 33 and NDK 26.3.11579264 (Tools -> Other -> NDK (Side by side) 26.3.11579264).  The former will end up in "C:\Program Files (x86)\Android\android-sdk" and the latter
  in "C\Program Files (x86)\Android\android-sdk\ndk\26.3.11579264" (or wherever your 'Program Files (x86)' folder is).
- In VS under Tools -> Options -> Cross Platform -> C++ -> Android, set Android SDK and Android NDK to the paths mentioned above.

Problems:
1. Going from NDK 25 to 26, 'lib64' in the path to clang was consolidated into the existing 'lib' folder.  This is a problem because if you go to <Edit>... under Configuration Properties -> VC++ Diretorie -> Include Directories in the project properties,
   you'll see that "$(LLVMToolchainPrebuiltRoot)\lib64\clang\$(LLVMVersion)\include" is an inherited value.  We have to uncheck 'Inherit from parent or project defaults', and manually add the new directory:  "$(LLVMToolchainPrebuiltRoot)\lib\clang\$(LLVMVersion)\include".
   This isn't really a problem, just more of an out-of-date inconvenience (and a time sink when it comes to figuring it out).

2. Going from NDK 25 to 26, "[NDK Root]\sources\cxx-stl\llvm-libc++" and "[NDK Root]\sources\cxx-stl\llvm-libc++abi" were removed.  These folders (which contain the C++ headers) get added as header search paths by default via $(StlIncludeDirectories),
   assuming you have 'Inherit from parent or project defaults' checked.  Immediately after these two folders get added, "$(SysRoot)\usr\include" gets added *automatically* (no matter what you do - you can't change this), which is where all the C libraries are.
   So, the order is correct:  C++ headers will be found first, and C headers second.  However, in NDK 26 where "sources\cxx-stl\llvm-libc++" and "sources\cxx-stl\llvm-libc++abi" are gone, you have to *manually* add
   "$(SysRoot)\usr\include\c++\v1" as a header search path instead so that the C++ headers can be found.  Alas, because you're specifiying it manually, it comes *after*
   the aforementioned-and-automatically-added C header path, "$(SysRoot)\usr\include".  Now there's a problem because the header search paths are ordered incorrectly - the C path is first and the C++ is second.
   And if you try and include a C++ header (say, #include <cstdlib>), you'll get this annoying-but-really-helpful error message:

   error : <cstdlib> tried including <stdlib.h> but didn't find libc++'s <stdlib.h> header.  This usually means that your header search paths are not configured properly. The header search paths should contain the C++ Standard Library headers before
   any C Standard Library, and you are probably using compiler flags that make that not be the case.

Notes:
- I added the -v switch to the clang compiler command via Configuration Properties -> C/C++ -> Command Line -> Additional Options in the project properties dialog to see the list of include paths in the Output window in the ordered that they'll be
  searched when you compile.  That really helped.
- I worked around this (hopefully temporarily) by renaming the "$(SysRoot)\usr\include" of my NDK 26.3.11579264 installation to "$(SysRoot)\usr\inc".
  This means that the default header search path addition will fail because the directory doesn't exist.  I can subsequently manually specify "$(SysRoot)\usr\inc" as a header search path AFTER I
  specific "$(SysRoot)\usr\inc\c++\v1" as one (along with "$(Sysroot)\usr\include\$(AndroidHeaderTriple)", to be complete).  Problem "solved"; everything builds and runs fine.
