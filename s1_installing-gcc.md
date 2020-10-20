# Installing GCC (and other GNU features) on Windows 7+

In this little tutorial, we'll be installing some UNIX programs by means of the "Minimal GNU for Windows" suite (**MinGW** for short) for Windows 7 and up. First, let's get **MinGW**.

1. Go to the [Sourceforge project page for MinGW](https://sourceforge.net/projects/mingw/), and download.
2. Install in a directory of choice (I personally chose `C:/Users/[name]/AppData/Local/MinGW`, to not have to install under `C:/` but to have it hidden somewhere with write privileges).
3. Browse to *Computer -> Properties -> Advanced -> Environmental Variables -> User Variables*, and edit the `PATH` variable by appending `;C:\Users\[name]\AppData\Local\MinGW\bin;`.

That's it for **MinGW**. We now have a command-line package installer for various UNIX toolkits. To install, for example, the GNU C Compiler (**GCC**): 

1. Open Windows's `cmd`.
2. Execute `mingw-get install gcc`.

That's it! For this course, you'll need some other utilities. You can get them by specifying each separately, or all at once as we've done below.
```
mingw-get install mingw-developer-toolkit msys-base mingw32-base
```

You are now all set up for compiling C on Windows! ~T.B.