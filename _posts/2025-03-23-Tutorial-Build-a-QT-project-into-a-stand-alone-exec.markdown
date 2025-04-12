---
layout: post
title:  "Tutorial: building a QT project to be a stand alone executable"
date:   2025-03-23 22:29:41 +0100
categories: tutorials
tags : ['Tutorial', 'QT', 'Desktop_app', 'PRT']
author : "Francesco Abate"
---

This is a guide from march 2024 about how to release a QT project as a standalone executable in windows, I created it in the context of the Polito Rocket team to find a way to use our ground station app (app we use to visualize data received from the rocket) as a stand alone executable, so that you do not need to copy a whole folder with libraries each time you pass it from a member of the club to another.

Being from a year ago some version might not be the freshest of each tool but at least this works 100%

> NOTE: the following guide is for **windows only**

### Deployable folder

The Windows deployment tool `windeployqt` is designed to automate the process of creating a deployable folder containing the Qt-related dependencies (libraries, QML imports, plugins, and translations) required to run the application from that folder.
Following the [official guide](https://doc.qt.io/qt-6/windows-deployment.html#the-windows-deployment-tool):

1. Switch to "Release" build type and build the project
2. Copy the generated `/path/to/release/build/folder/appground-station.exe` file into a new folder
3. Open the cmd in the `bin` directory inside the Qt directory in your local machine (usually `C:/Qt/<version>/<compiler>/`)
4. Run `windeployqt /path/to/the/executable/you/copied.exe --qmldir /path/to/release/build/folder/ground-station`

> NOTE: the deployable folder for some reason does not include some Qt dlls required to run the application. The person you are giving this folder to still needs to provide those dlls.

### Static executable

It is possible to build statically the application and produce a single executable. You will need the following tools installed on your machine and present in the `PATH` variable:

- CMake & git (already needed for development)
- [MSVC](https://visualstudio.microsoft.com/vs/features/cplusplus/) (Microsoft Visual C++) compiler
- [Perl](https://www.perl.org/get.html)

Before building our application, you first need to statically buil Qt. Open a command prompt with admin rigth and perform the following:

> Here we use `C:\Qt` as installation folder of Qt, but actually you can use whatever folder you like: make sure it already exists

```bash
#  Download the git Qt project
git clone git://code.qt.io/qt/qt5.git qt6
cd qt6
git switch 6.4.3

# init the repository with the qt modules used
perl init-repository --module-subset=qtbase,qtshadertools,qtdeclarative,qtcharts,qtserialport

# create the build directory
cd ..
mkdir qt6-build
cd qt6-build

# configure and build
..\qt6\configure.bat -static -release -opensource -confirm-license -prefix C:\Qt
cmake --build .
cmake --install .
```

Now add `CMAKE_PREFIX_PATH` to your environment variables (**not** to the `PATH` variable) with the value `C:\Qt` (or whatever folder you chose for the installation folder of Qt). If you have other cmake projects, you can remove/restore this variable after completing the build of the application.

Finally, create the folder where you want to build the ground-station project, open the Native Tools Command Prompt (it is part of Visual Studio and can be found using the windows search bar) and run the following commands:

```bash
cd <newly\created\ground-station\build\folder>
cmake <path\to\ground-station\source\folder> -G "Visual Studio 17 2022" -DCMAKE_BUILD_TYPE=Release
msbuild /m /p:Configuration=Release ground-station.sln
```

If you have older versions of VS2022, replace the string with the version you have (e.g. `"Visual Studio 16 2019"`). At the end of the operation the message `BUILD FAILED` will be displayed, but that is **not** true. You should now have an executable inside your build folder.

> Guide based on : [Building Qt 6 from Git](https://wiki.qt.io/Building_Qt_6_from_Git)
