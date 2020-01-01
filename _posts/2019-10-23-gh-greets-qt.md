---
layout: post
title: GitHub Actions, the missing notes
subtitle: CMake, Qt and IFW
gh-repo: skypjack/gh-greets-qt
gh-badge: [star, follow]
tags: [gh, gh-actions, qt, ifw]
---

GitHub Actions is a truly nice tool. Since I tried it the first time, I can no
longer do without it... and it's still only available in public beta!<br/>
However, it has some shortcomings, both in terms of packages and documentation,
which make it difficult to use in some cases. This is because it's _young_, I
don't blame the tool but we still have to face the problem.

With these notes I hope to help _another me_ who's about to compile a Qt
software and create an installer on the new service offered by GitHub.

## Introduction

Long story short, this post is nothing but a set of notes about compiling a Qt
project based on cmake and creating an installer via IFW (or rather CMakeIFW) on
the Windows platform.

Why Windows? Mainly because Windows was the one that gave me the most trouble in
many ways. If you understand how to do it on this platform, you understand how
to do it on all the others.<br/>
Also, keep in mind that on Linux and MacOS you can install the Qt libraries via
_official_ channels (eg `apt`), even if going through the installer frees you
from the constraints on the packaging of the specific distribution. In general,
I would recommend the use of the official installer rather than the system
packages and, in this case, these notes will help even on those platforms.

I don't pretend to have found the smartest nor best solution ever, but at least
I've the CI up and running now and it creates an installer whenever I cut a new
tag. The installer is then published directly in the GitHub releases, where my
users can safely download it having access to the repository.<br/>
Not bad at least as a starting point.

### GitHub Actions greets Qt, CMake and IFW

With this post I've also published (or rather, I made public) a new
[repository](https://github.com/skypjack/gh-greets-qt) on GitHub (which I will
**NOT** update over time, in case there were doubts).<br/>
It's a _template project_ that can be used to bootstrap a CMake/Qt based
software, if you want to compile it using GH Actions. At the very least, it can
serve as an inspiration.

I apologize if the project may seem imperfect or if it contains apparently
superfluous parts. I extracted the content from a real world software and tried
to clean it of everything that wasn't necessary.<br/>
Unfortunately (or fortunately, it depends on the point of view), the original
repository is private and contains code that I cannot publish because I'm under
contract. Therefore it wasn't possible to use it as an example, although I would
have liked to do it.

I hope I haven't made too many mistakes in preparing a minimal repository that
can make clear what I'm about to tell.

### Headless Installer

Unfortunately, the Qt installer doesn't have a headless mode. We must find a
workaround to face this limitation if we want to run it silently.<br/>
It's not that hard actually. In fact, the Qt installer is based on the _Qt
Installer Framework_ (also known as
[Qt IFW](https://doc.qt.io/qtinstallerframework/index.html)) and it's therefore
_scriptable_.

In short, when you run the installer, you can pass a file containing a script
that will simulate the user and his/her clicks on the various _Ok_, _Next_,
_Accept_ buttons and so on.<br/>
This becomes a little more difficult when it comes to selecting the components
you want to install, but not so hard.

[This](https://doc.qt.io/qtinstallerframework/noninteractive.html) is the
documentation to follow if you want to write your own custom script.<br/>
At the end of the day, your file will contain a `Controller` that exposes a
bunch of functions that will be called by the installer itself. As an example:

```
Controller.prototype.WelcomePageCallback = function() {
    gui.clickButton(buttons.NextButton, 5000);
}
```

This helps to simulate a click on the _Next_ button with a delay of 5 seconds,
nothing less and nothing more. It's not the user but behaves as if it were.<br/>
The delay is there because the welcome page takes some time to contact the
server and only at the end will enable the button for the user, so we worry
about simulating also the wait.

Another interesting example is the callback for the component selection page:

```
Controller.prototype.ComponentSelectionPageCallback = function() {
    var selection = gui.pageWidgetByObjectName("ComponentSelectionPage");
    gui.findChild(selection, "Latest releases").checked = true;
    gui.findChild(selection, "FetchCategoryButton").click();

    var widget = gui.currentPageWidget();
    widget.deselectAll();

    widget.selectComponent("qt.qt5.5131.win64_msvc2017_64");
    widget.selectComponent("qt.tools.ifw.31");
    widget.selectComponent("qt.tools.openssl");
    widget.deselectComponent("qt.license");
    widget.deselectComponent("qt.installer");

    gui.clickButton(buttons.NextButton);
}
```

In our case, we are interested in installing the latest release from Qt and
therefore we have to check the right box in the category section. Once this is
done, a button will be pressed (from script, of course) that will force the
refresh of the list of available components. Then, from the latter we will
select and install the components of interest and press _Next_.<br/>
Sounds simple, doesn't it?

Sadly, the (little) documentation I found online isn't up to date and writing a
complete script can be frustrating. The good news is that it can easily be done
locally, with the installer for our system.<br/>
To facilitate the task, the repository associated with this post contains also a
[script](https://github.com/skypjack/gh-greets-qt/blob/master/ci/qt.qs) that is
complete and working when I'm writing this. You'll only need to update it when
necessary, from now on.

### CMake & CPack

As for `CMake`, there's not much to say. If you know its syntax, nothing special
is required for compilation under Windows and then via GH Actions.<br/>
The same isn't true for `CPack` though. This isn't due to GH Actions however,
but rather to Qt in general and Qt IFW in particular.

When we prepare an installer, we don't want to install **only** our executable.
Instead, we need to deploy also the set of libraries required to make it run.
To do that, the [`windeployqt`](https://doc.qt.io/qt-5/windows-deployment.html)
executable comes in handy:

>`windeployqt` takes an `.exe` file or a directory that contains an
>`.exe` file as an argument, and scans the executable for dependencies.

The only thing we have to do is to run it under the hood by means of `CPack`,
then _install_ (as in `CPack` command `install`) what it returns:

```
set(CPACK_GENERATOR IFW)

include(CPack REQUIRED)
include(CPackIFW REQUIRED)

# ...

if(CMAKE_BUILD_TYPE MATCHES Release)
    set(BINARIES_TYPE --release)
else()
    set(BINARIES_TYPE --debug)
endif()

add_custom_command(
    TARGET gh-greets-qt POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_BINARY_DIR}/windeployqt_stuff
    COMMAND $ENV{QTDIR}/bin/windeployqt.exe ${BINARIES_TYPE} --compiler-runtime --dir ${CMAKE_BINARY_DIR}/windeployqt_stuff $<TARGET_FILE:gh-greets-qt>
)

install(
    DIRECTORY ${CMAKE_BINARY_DIR}/windeployqt_stuff/
    DESTINATION gh-greets-qt
    COMPONENT gh-greets-qt_component
)
```

This is a simplified excerpt from the
[`CMakeList.txt`](https://github.com/skypjack/gh-greets-qt/blob/master/CMakeLists.txt)
file of the accompanying project. For the sake of completeness, I must say that
I've used the
[CMake IFW Generator](https://cmake.org/cmake/help/v3.14/cpack_gen/ifw.html) for
my purposes. Here is a quote from its documentation:

>CPack IFW generator prepares project installation and generates
>configuration and meta information for Qt IFW tools.

It's not strictly necessary but it makes life easier when you want to make Qt
IFW and CMake work together.<br/>
It's quite used and pretty solid, in the past I've also had the chance to
contact the author to help him correct some bugs, a person who proved to be very
helpful and determined to improve this module more and more. So, I find it worth
a try.

That said, in the snippet above we're literally asking `windeployqt` to fill the
`windeployqt_stuff` directory with all is needed to make our executable run
correctly. We don't care much of what it will put in this directory because we
_trust_ the tool. However, something in particular deserves a special mention.

From the previous section, you saw that we installed the
`qt.qt5.5131.win64_msvc2017_64` component. It will result in `windeployqt`
putting in the bundle the `vc_redist.x64` executable.<br/>
This is particulary important and brings us directly to the next section and to
yet another script. However, this time the purpose is to _extend_ our installer
rather than turning the Qt one into a non-interactive tool.

### The Component and the Visual C++ Redistributable

By using CMake IFW we can easily setup one or more Qt IFW
[`Component`s](https://doc.qt.io/qtinstallerframework/scripting.html) in the
form of some scripts to use to _extend_ an installer. Usually, these scripts are
aimed to create menu shortcuts or things like that:

```
if(systemInfo.productType === "windows") {
    // ...

    component.addOperation(
        "CreateShortcut",
        "@TargetDir@/gh-greets-qt/gh-greets-qt.exe",
        "@StartMenuDir@/Hi-Qt.lnk",
        "workingDirectory=@TargetDir@"
    );
}
```

However, we can use the same tool for everything. In our case, we want to use it
to execute the Visual C++ Redistributable executable so as to install what is
needed to run our software on the user system. In fact, it will install runtime
components required by the applications compiled with Microsoft Visual Studio,
that is exaclty our case:

```
if(systemInfo.productType === "windows") {
    component.addElevatedOperation("Execute", "{0,3010,1638,5100}", "@TargetDir@/gh-greets-qt/vc_redist.x64.exe", "/quiet", "/norestart");

    // ...
}
```

Unfortunately, we cannot do run this executable as a normal user. That's why
this time we use `addElevatedOperation` rather than `addOperation` (refer to the
official documentation for Qt IFW for further details on these functions).<br/>
What's the difference? Long story short, Windows will warn the user that we are
going to do something risky and therefore we need higher privileges to proceed.
If you're a Windows user, you know what I'm talking about: that's the _Ok_
button you press usually between an _uff_ and a _damn it, again?_.

As a side note, you'll not need to take this step if you decide to compile your
software using `mingw` or any other alternative. However, it was one of the
steps I spent the most time on and I found it interesting to mention this
explicitly rather than hurry up with an _you can look into it and figure it out
for yourself_.

### GitHub Workflow

Finally, it's time also for GH Actions. This is the simplest part in a sense, so
simple and user-friendly this tool is.

I didn't spend much time to make the example project configuration as flexible
as possible. Rather, I tried to write it so that it was simple to understand and
to modify. At the very least It case serve as a basis for your project.<br/>
I don't know how confident you are with the syntax of this tool, but I think
it's simple enough to be understood in large part at first glance:

```
jobs:
  windows:
    runs-on: windows-2019

    steps:
    - name: Checkout
      uses: actions/checkout@v1
    - name: Prepare
      working-directory: build
      run: |
        curl -vLo qt-unified-windows-x86-online.exe http://download.qt.io/official_releases/online_installers/qt-unified-windows-x86-online.exe
        qt-unified-windows-x86-online.exe --verbose --script ..\ci\qt.qs
    - name: Configure
      working-directory: build
      run: cmake -DCPACK_IFW_ROOT=Qt/Tools/QtInstallerFramework/3.1 -DCMAKE_BUILD_TYPE=Release -G"Visual Studio 16 2019" ..
    - name: Compile
      working-directory: build
      run: cmake --build . --config Release -j 4
    - name: Package
      working-directory: build
      run: cmake --build . --config Release --target package
```

This is a simplified exceprt from the
[`build.yml`](https://github.com/skypjack/gh-greets-qt/blob/master/.github/workflows/build.yml)
file of the accompanying project.<br/>
We have only one job, the goal of which is to compile our project for Windows
and prepare an installer.

The first step is obviously the checkout of the project. Next, the Qt installer
is downloaded and executed in non-interactive mode.<br/>
Do you remember the script mentioned above? This is where it comes into play and
does its dirty work.<br/>
The third step executes the configuration of the project, which can be
summarized as _launches `CMake` and that's all_. Finally, we build the
executable and create the package.

As you can see, much of what is needed to have everything up and running is
contained in the files mentioned in the previous sections. Therefore, our
workflow does nothing more than download the installer and launch the programs
we need to get the job done.<br/>
However, there is a question that may arise now: once the package is created,
how is it retrieved?

It would be great if our installer was published directly between the releases
available in the GitHub project.<br/>
In this we are helped by the fact that anyone can write an _action_. There are
already dozens of them on the web and they exist for every taste, without
necessarily having to reinvent the wheel every time.<br/>
In the specific case, for the project related to this post, I used
[this](https://github.com/softprops/action-gh-release) action available directly
on GitHub. Its goal is simple:

>GitHub Action for creating GitHub Releases

And using it is just as simple, fortunately:

```
- name: Release
  uses: softprops/action-gh-release@v1
  if: startsWith(github.ref, 'refs/tags/')
  with:
    files: build/gh-greets-qt_installer.exe
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

The way I configured it and thanks to the tools provided by GitHub, this runs
only when a tag is created instead of for each push.<br/>
When this happens, I'll find a new release with the `gh-greets-qt_installer.exe`
file published directly online, without having to worry anymore. Nothing easier
and definitely convenient.

## Conclusion

All in all, compiling your project and creating a Windows installer with IFW via
GH Actions wasn't all that difficult. It's mainly a matter of putting the pieces
together from previous experiences.<br/>
Unfortunately, however, if you have no experience at all with this kind of
tools, I admit that it can be a little tricky.

For this reason, I hope that these notes can help those users who are taking the
first steps in this direction. Maybe this post won't tell you everything but I'm
sure it will tell you enough to make you create your _workflow_.

## Let me know that it helped

I hope you enjoyed what you've read so far.

If you liked this post and want to say thanks, consider to star the
[GitHub project](https://github.com/skypjack/skypjack.github.io) that hosts this
blog. It's the only way you have to let me know that you appreciate my work.

Thanks.
