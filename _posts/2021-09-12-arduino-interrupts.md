---
title: Arduino Timer Interrupts
author: Joe Post
date: 2021-09-12 14:10:00 +0800
categories: [Blogging, Tutorial]
tags: [c++, programming, arduino, interrupts]
---

## Foreword

In this blog, we're going to explore a fundamental aspect of the Arduino architecture: the timer interrupt. Interrupts are an entire topic in and of themselves, but today we're focusing specifically on timer interrupts. This is for several reasons:

1. Timer interrupts are a subset of interrupts and should be logically extendable to other interrupts,
2. Timer interrupts can provide a very useful, precision time measurement utility for a variety of applications, and
3. I happen to find timer interrupts particularly interesting since they involve fundamental portions of the Atmel architecture.

This blog assumes you have an Arduino board (I used an Arduino Due), and the Arduino software installed on your computer. I will not be covering the exact steps to compile and upload the code to your board; it is assumed that you have done this before. 


## Timer Architecture
Before we dive into the code, we need to understand what _exactly_ what the timer infrastructure is within the Atmel chipset of interest. Again, I'm using an Arduino Due, but all you need to do is consult your specific chiset's user guide to find out the equivalent registers and code. 

The Arduino Due uses an Atmel SAM3X8E processor, which is much more capable than the Atmega-based line of other Arduino boards. The processor uses an 84 MHz clock, uses a Thumb-2 instruction set architecture (ISA) subset consisting of all base Thumb-2 instructions, 16-bit and 32-bit, a Harvard processor architecture enabling simultaneous instruction fetch with data load/store, three-stage pipeline, single cycle 32-bit multiply, hardware divide, and several other features.


The SAM3X83 has several internal timer/counter (TC) modules capable of generating interrupts. Page 38 of the [datasheet](https://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-11057-32-bit-Cortex-M3-Microcontroller-SAM3X-SAM3A_Datasheet.pdf) documents the following peripheral identifiers related to the timer module interrupts:

| *Instance ID* | *Instance Name* | *NVIC Interrupt* | *PMC Clock Control* | *Description* |
| ------------- | ------------- | ------------- | ------------- | ------------- | 
| 27 | TC0 | X | X | Timer Counter Channel 0 | 
| 28 | TC1 | X | X | Timer Counter Channel 1 | 
| 29 | TC2 | X | X | Timer Counter Channel 2 | 
| ... | ... | ... | ... | ... | 
| 35 | TC8 | X | X | Timer Counter Channel 8 | 

Ok there are quite a few new acronyms above. Let's break these down. The TC modules each manage some sort of timing functionality - they can count up, down, issue interrupts, toggle general purpose input/outputs (GPIOs). These interrupts and associated peripheral identifiers are used by the Nested Vector Interrupt Controller (NVIC) to manage interrupts. The NVIC provides up to 16 interrupt priority levels. 



As far as system-level views of interrupts are concerned, there are some core registers that are associated with them.



## Let's Go to the Code!
Ok, let's finally jump in to the code. 

```c++
#include <iostream>

int main() {
	std::cout << "Hello, world!" << std::endl;

	return 0;
}
```
So hopefully some of these concepts look vaguely familiar. We have an ```#include``` indicating that we want to include a built-in library. We know it's built-in since we're using ```<>``` brackets and not quotations. For non-standard libraries (e.g., third party libraries, other source files you write), you'll typically indicate these with quotations (e.g., ```#include "my_lib.hpp"```).

While _technically_ C++ codes doesn't have to return an int, I've never not returned an integer exit code. From my perspective it is sound programming habit to do so. In addition, your code must include a ```"main()"``` function. See [this](https://en.cppreference.com/w/cpp/language/main_function) reference for more details. This tells the compiler where the main entry point into the program is. Now in this case, we didn't pass any arguments to the program. We can just as easily do that via:

```c++
#include <iostream>

int main(int argc, char* argv[]) {
	if (argc > 0) {
		std::cout << "Program received " << argc << " arguments" << std::endl;
		std::cout << argv[1] << std::endl;
	} else {
		std::cout << "Hello, world!" << std::endl;
	}
	return 0;
}
```

Once compiled into an executable on a Windows machine, say "main.exe", you can execute it on the command line using ```main.exe Arg1``` or simply ```main.exe```. Note that ```argv[]``` always contains the program name at the zeroth location. Therefore, any user-specified inputs are available at ```argv[1]``` and beyond. 

## Compilation
Once written, you'll need to compile your code to create an executable. Technically there are two high-level processes that need to occur:
1. Compilation: Actually converting source code into binary, machine-readable code called objects files, and
2. Linking: linking the various object files together to form a final executable.

However, most of the time you'll execute both of these simultaneously. In order to compile (meaning, compile and link in this case) the above file, say main.cpp, use the following:

```
g++ main.cpp -o main.exe
```

The above assumes we're working on a Windows machine, and says "hey g++, take my file "main.cpp" and compile and link to form an executable output file (hence the -o option) called "main.exe"". You should see the executable file show up in your working directory. If you're running on a linux box, you'll need to execute:

```
g++ main.cpp -o main
```

Note that we simply removed the trailing extension to get this to work on our Linux box. Hopefully it's obvious how manually specifying compilation arguments can get out of hand rather quickly. In order to support compilation on multiple operating systems, it typically behooves us to write Makefiles. We'll talk about that in a second. First, it should be noted that the resulting executable is only executable on machines with your CPU architecture, which is rather unlike languages like Java (which run in virtual machines) and Python (which is ran using an interpreter). You an "cross-compile" an executable to run on another OS/architecture, but that is outside the scope of this tutorial. 

## Makefiles
Makefiles are wonderful little files that help manage the compilation of code more efficiently. While you don't need Makefiles in your workflow, they _significantly_ aid in creating portable code that can be downloaded on any machine and compiled successfully. Let's look at a simple Makefile:

```Makefile
# unfortunately, tabs are really important in Makefiles. only tab where indicated,
# and do not use spaces from the left at all. Also, don't use spaces around the "=" sign.
CC=g++

# can include additional source files here as needed. For example:
# OTHERS=helpers.cpp
# for now, just leave empty
OTHERS=

# if we're on a linux box, we don't need to.
# -Wall means "display all compilation and linking Warnings"
# you can add any number of compilation flags here, separated by a space. 
CC_FLAGS=-Wall

# for Windows operating system, need to manually specify the file extension.
ifeq ($(OS),Windows_NT)

# windows likes .exe files
EXEEXT=.exe

else
# do something different here if needed.
endif

# specify that, when this Makefile is called, execute all of the functions listed here
all: main_file

main_file:
	# now create the actual compile command.
	# IMPORTANT: make sure you 'tab' here and not four-space!!!
	${CC} main.cpp ${CC_FLAGS} ${OTHERS} -o main${EXEEXT} 

# now specify some clean-up actions that can optionally, manually be executed. 
clean:

ifeq ($(OS),Windows_NT)
	del main${EXEEXT}
else
	rm main
endif
```	

Hopefully the above comments are straight forward. You can add as many actions as you'd like in case you're building multiple executables. As you can see, Makefiles scale quite nicely. You'll need to install "make" in order to run the above. Once installed, simply enter ```make``` on the command line from within the same directory as the Makefile and it should run the 'all' actions without issue. Use ```make clean``` to execute the cleanup actions above.

## Third Party Libraries
I have used a really neat third party library called [Boost](https://www.boost.org/) many times before. It is really more of a collection of source code written by various people and organizations around the world. There is a lot of useful libraries in there, from multithreading to image processing. Let's say you want to include Boost Generic Image Library (GIL) code, which includes a multiple of image manipulation and processing tools. You'll need to download the source code, know the location of that folder, and edit main.cpp as follows:

```c++
#include <iostream>
#include "boost/gil/extension/io/png/old.hpp" // defines "png_read_dimensions" function
#include "boost/gil/point.hpp" 	// defines "point" class type

int main(int argc, char* argv[]) {
	// ensure a filename is provided
	if (argc <= 0) {
		std::cout << "No filename specified, exiting." << std::endl;
		return 1;
	}

	// obtain the dimensions of the png file
	point<std::ptrdiff_t> dims = png_read_dimensions(argv[1]);

	// print some debug info
    std::cout << "Image is of height: " << dims.y << " pixels and width: " << dims.x << " pixels." << std::endl;
	
	return 0;
}
```

NOTE: The above code will break if you run it and provide the name of non-existent image file. 

You'll notice the "<>" characters surrounding the "std::ptr_diff_t" declaration, as well as the "::" opertors therein. These are concepts known as "templating" and "namespace resolution" respectively. These concepts are outside the scope of this tutorial, but just know they are at the core the usefulness of C++.

Now, to compile this, we need to tell the compiler where the source files are for the "point" class and "png_read_dimensions" function. This is a little counter-intuitive since we specify them in the header of main.cpp, however the compiler may not know _exactly_ where those files are, since we only specified the relative path of the source files from within the "boost" folder. Therefore, we have to provide the base path to the boost folder. The resulting Makefile therefore looks like:


```Makefile
# unfortunately, tabs are really important in Makefiles. only tab where indicated,
# and do not use spaces from the left at all. 
CC=g++

# if we're on a linux box, we don't need to.
# -Wall means "display all compilation and linking Warnings"
# you can add any number of compilation flags here, separated by a space. 
CC_FLAGS=-Wall

# for Windows operating system, need to manually specify the file extension.
ifeq ($(OS),Windows_NT)

# windows likes .exe files
EXEEXT=.exe

# define base path for boost libs, for example:
BOOST=D:\\dev\\boost

else
# also define base path for boost library, for example:
BOOST=/usr/lib/boost
endif

# specify that, regardless, execute all of the functions listed here
all: main_file

main:
	# now create the actual compile command.
	# IMPORTANT: make sure you 'tab' here and not four-space!!!
	# Use the -I flag to find any #include files that are not part of the language
	${CC} main.cpp ${CC_FLAGS} ${OTHERS} -I${BOOST} -o main${EXEEXT} 

clean:

ifeq ($(OS),Windows_NT)
	del main${EXEEXT}
else
	rm main
endif
```	

And there you go! Run 'make' from the command line from within the folder containing the Makefile and main.cpp, and you should be able to run your new executable using the following for linux:

```
main your_png_file
```

There are many, many different Makefile techniques you can use. For example, you may wish to separately compile separate object files in a unique way prior to compiling multiple object files together, instead of simply specifying the compilation of a single executable. [This](https://www.cs.fsu.edu/~lacher/lectures/Output/makefiles/tsld002.html) website has a few decent, simple examples that demonstrate this capability. 

## Conclusion
We covered quite a few topics in this intro to C++ and general workflow. There are so, so many more interesting concepts to cover such as polymorphism, templates, threads, and many more. I may make more tutorials on these topics in the future. As usual, thanks for reading!

## Learn More

For more knowledge about Jekyll posts, visit the [Jekyll Docs: Posts](https://jekyllrb.com/docs/posts/).

