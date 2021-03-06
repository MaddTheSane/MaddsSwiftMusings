# Swift in Makefiles, Madd's take

### Introduction

I've noticed that a lot of Makefiles that compile Swift code use the Xcode build structure. For some people, this isn't desirable. They also don't take into account if you have source code for other languages, or if you need to get Swift classes into Objective C. When looking for how to get Swift compiling for [NetHack 3D](https://github.com/MaddTheSane/nh3d_OSX) in a Makefile, I decided to use [the others](http://owensd.io/blog/swift-makefiles---take-2/) as a starting point to make my own, as well as taking cues from Xcode.

The file that I mention is in *[sys/Cocoa_GNUStep/Makefile.src](https://github.com/MaddTheSane/nh3d_OSX/blob/b615ae48214338ab30008fdb1a04f15f9f41c823/sys/Cocoa_GNUStep/Makefile.src)*, and is moved to the proper place when the `setup.sh` file is run. This is only so I can generate all the needed content for the NetHack side of things, as I usually use Xcode to compile and code.

This only covers Makefiles building Swift programs for OS X.

### Makefile definitions

First, I have all the Swift files as a seperate define from the C and Objective C files. I will talk a bout why later. I also distinguished the object files of Swift from the rest, with a define including them with the rest of the object files (Or at least those used by the NetHack3D port).

	WINNH3DSSRC = ../win/nh3d/GLHelpers.swift ../win/nh3d/SwiftNetHack\ bridge.swift \
		../win/nh3d/MapModel.swift ../win/nh3d/NH3DOpenGLViewSwift.swift \
		../win/nh3d/NH3DPreferenceController.swift ../win/nh3d/NH3DMessaging.swift
	[...]
	WINNH3DSOBJ = GLHelpers.o SwiftNetHack\ bridge.o MapModel.o NH3DOpenGLViewSwift.o \
		NH3DPreferenceController.o NH3DMessaging.o
	WINNH3DOBJ = NH3DMapView.o NH3DMenuItem.o NH3DMenuWindow.o \
		NH3DModelObject.o NH3DPanel.o \
		NH3DTileCache.o NH3DUserMakeSheetController.o NH3DUserStatusModel.o winnh3d.o tile.o \
		$(WINNH3DSOBJ) NH3DMapItem.o

*Way* later in the Makefile, I define the flags used by Swift:

	SWIFTFLAGS = -module-name $(GAME) -import-objc-header \
	../win/nh3d/NetHack3D-Bridging-Header.h -I../include \
	-sdk $(shell xcrun --show-sdk-path -sdk macosx)


Let's take you through what the flags do:

* `-module-name` marks the Swift module name. Here, I set it to what the game's name is.</li>
* `-import-objc-header` is the bridging header imported by Swift. Here, it's a bridging header used by the Xcode project. This specific header is perhaps *not* what you want to do, as it contains functions in itself (NetHack uses a lot of macros. Swift doesn't like macros more complex than `#define HELLO "hi"`.)
* `-I` sets the include path, specifically for C headers.
* `-sdk` is needed to point to the OS X SDK, otherwise system headers/frameworks can't be found. 

Following this, there's a few make rules. I'm changing the order they come in so I can talk about them better.

### Makefile Swift rule
First, a Swift compile rule:

	GLHelpers.o: $(WINNH3DSSRC) ../win/nh3d/NetHack3D-Bridging-Header.h $(HACK_H)
		swiftc -frontend -c -color-diagnostics -primary-file ../win/nh3d/GLHelpers.swift \
		$(filter-out ../win/nh3d/GLHelpers.swift,$(WINNH3DSSRC)) $(SWIFTFLAGS) \
		-emit-module -emit-module-path GLHelpers-partial.swiftmodule -o GLHelpers.o </code>

Taking it apart:

* `GLHelpers.o:` is a standard name for compiled code.
* `$(WINNH3DSSRC) ../win/nh3d/NetHack3D-Bridging-Header.h $(HACK_H)` are the requirements for the source file. I decided to have all the Swift sources (I said that **WINNH3DSSRC** had a reason), the bridging header, and something from the NetHack side of things. If any of these files are modified, the make rule is ran again.
* `swiftc` is the Swift compiler
* `-frontend` tells Swift to only use the front end. Don't know how critical it actually is...
* `-c` ...Not really sure what this does, or if it's needed.
* `-color-diagnostics` outputs colored text to the terminal if there's a warning/error in compiling.
* `-primary-file ../win/nh3d/GLHelpers.swift` is the source file we're compiling. This is needed because...
* `$(filter-out ../win/nh3d/GLHelpers.swift,$(WINNH3DSSRC))` ...we need to include the other Swift source files as well, as Swift has no headers. This does not compile the other sources; it just lets Swift know about other functions/classes. The **$filter-out** macro is to make sure we don't include the file we're compiling
* `$(SWIFTFLAGS)` The Swift flags.
* `-emit-module` tells Swift to create a Swift module. This will be needed if you need the Swift classes in Objective-C, or are compiling a Swift framework.
* `-emit-module-path GLHelpers-partial.swiftmodule` points to the location of where we want the created module to be. Suffixing the file with *-partial* is only a convention, not a requirement (Xcode uses *~partial*).
* `-o GLHelpers.o` tells Swift where to put the compiled code. This file can be linked like any *.o* generated by any other language.

And I did that for each Swift source file. Inelegant, but it works. A better way of doing it could probably be:

	%.o: ../win/nh3d/%.swift $(WINNH3DSSRC) ../win/nh3d/NetHack3D-Bridging-Header.h $(HACK_H)
		swiftc -frontend -c -primary-file ../win/nh3d/$@ \
		$(filter-out ../win/nh3d/$@,$(WINNH3DSSRC)) $(SWIFTFLAGS) \
		-o $*.o -emit-module -emit-module-path $*-partial.swiftmodule

###Swift Module

The following combines the Swift modules into one. It is only needed if you need to get a header to use the Swift classes in Objective-C, or if you're building a Swift library/framework.

	NetHack3D.swiftmodule: $(WINNH3DSOBJ)
		swiftc -frontend -c -color-diagnostics -emit-module $(SWIFTFLAGS) -emit-module-path \
		NetHack3D.swiftmodule $(subst .o,-partial.swiftmodule,$(WINNH3DSOBJ)) -parse-as-library

Flag explanations:

* `NetHack3D.swiftmodule: $(WINNH3DSOBJ)` the make rule for the combined Swift module. We depend on the compiled Swift sources, but we rely on the generated Swift modules.
* `swiftc -frontend -c -color-diagnostics -emit-module $(SWIFTFLAGS)` is just the same as when we compiled the Swift sources.
* `-emit-module-path NetHack3D.swiftmodule` tells Swift where to put the generated module. As a general rule, make sure it matches the module name.
* `$(subst .o,-partial.swiftmodule,$(WINNH3DSOBJ))` imports the Swift modules from the compiled Swift source code for use. The *subst* macro converts the names of the object files in **WINNH3DSOBJ** to the Swift module path.
* `-parse-as-library` tells Swift to treat the imported files as Swift modules, as opposed to source code.

### Bridging Header generation

	NetHack3D-Swift.h: NetHack3D.swiftmodule
		swiftc -frontend -c -parse-as-library -color-diagnostics $(SWIFTFLAGS) \
		NetHack3D.swiftmodule -emit-objc-header-path NetHack3D-Swift.h

Tag info:

* `NetHack3D-Swift.h: NetHack3D.swiftmodule` is the make rule for the generated Objective-C header. We only depend on the previously generated module file.
* `swiftc -frontend -c -color-diagnostics $(SWIFTFLAGS)` is just the same as when we compiled the Swift sources. `-emit-module` isn't used because we're not building a module.
* `NetHack3D.swiftmodule` is where we import the module
* `-emit-objc-header-path NetHack3D-Swift.h` tells Swift to generate the header and where to put it.

Also, any code that imports the generated header **must** have the generated header as a make dependency, otherwise `make` might try to compile the source code before the header is generated and throw an error.

###Linking

In order to link the Swift source code properly, additional flags need to be added to the linker:

	WINNH3DLIB = [...] \
	-L/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/swift/macosx/ \
	-rpath /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/swift/macosx/ \
	-rpath @executable_path/../Frameworks

linking flags:

* `-L /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/swift/macosx/` is needed to point to the needed libraries used by swift, including the Objective-C shims/helpers. What I have here is fragile, as any number of things may break this. 
* `-rpath /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/swift/macosx/` helps when running the app, pointing the compiled app to search that path for the linked libraries specified. **Do not use this for distribution** as your end user might not have Xcode installed, or even the same version that you're compiling. As noted earlier, I use Xcode to build this project, which handles copying the Swift libraries for me.
* `-rpath @executable_path/../Frameworks` tells the app to look for libraries in the application's *Frameworks* subdirectory. *This* is where you would put the needed Swift libraries, and what Xcode automatically adds when you add Swift code to existing projects.