---
layout: post
title: Package PysimpleGUI scripts as OSX APP files with PyInstaller
---

PySimpleGui is a great python library that allows users to create a simple yet effective GUI for their python script with just a few lines of code (if you haven't already, [you should really check it out](https://pysimplegui.readthedocs.io/en/latest/)). The only problem is that if you your plan is to distribute your GUI app to users, a Python script may not be the most efficient way: different python versions and missing packages are often a good recipe for errors, especially because GUI apps are often aimed at users who don't necessarily have the knowledge to debug these kind of problems. Luckily there is a tool to make the whole distribution process a lot easier for everyone: introducing PyInstaller (again, if you haven't already, [go check it out](https://www.pyinstaller.org)). PyInstaller will package (freeze) your Python script into a stand-alone executable, turning it into a nice bundle that can be easily distributed on virtually any platform (Windows, GNU/Linux, Mac OS X, FreeBSD, Solaris and AIX). Not only does PyInstaller make the distribution process simpler, but it also copies all the imported modules and any other resources that your script needs inside of the executable. This essentially means two things: one is that you get to choose which Python version and modules to package and two is that users won't ever have to deal with installing dependencies or debugging errors caused by different Python versions.

Now, packaging your Python script on Windows is well documented in the [PySimpleGui documentation](https://pysimplegui.readthedocs.io/en/latest/), but going through the same process on OS X is another story. This is why I have decided to write a full tutorial on how to do this.

### Example 1 - No additional data

Let's take this simple "Hello World!" script as an example:

```python
import PySimpleGUI as sg # Import PySimpleGui module
from sys import exit # from * import statement
import random # impot * statement

layout = [[sg.Text("Hello World! " + str(random.randint(0, 10)), font=("Helvetica", 30), key="output")]] # Define GUI layout

window = sg.Window("Hello World!", layout) # Initialize window

while True: # Start event loop
    event, values = window.Read()

    if event is None: # Quit
        window.Close()
        exit(0)
```

As you can see this script will open a window containing only the string "Hello World!" and a number from 0 to 10. This might seem like a pointless application, but in fact by packaging this script we can test if the `import` statements work (i.e if the modules are packaged alongside the script). Note that in this case, the script does not require any additional data (we will cover that case later).

To package such a Python script into an OS X APP bundle, execute the following command:

```bash
pyinstaller --onefile --windowed --add-binary='/System/Library/Frameworks/Tk.framework/Tk':'tk' --add-binary='/System/Library/Frameworks/Tcl.framework/Tcl':'tcl' <program_name.py>

```

Let's understand what is going on:

* The `--onefile` flag specifies that a GNU/Linux executable should be created. This is required to create an OSX APP
* The `--windowed` flag specifies that an OS X APP should be created
* The `--add-binary` flags specify that Tkinter and Tcl should be included. These executables are required for PySimpleGUI to work and PyInstaller **WILL NOT INCLUDE THEM AUTOMATICALLY**
* `<program_name.py>` is your scripts filename

Let's execute the command and after a few seconds our APP seems to have been created.

![example-1-compiled](/images/posts/2019-8-19-Bundle-PySimpleGui-scripts-as-OSX-APP-files-with-PyInstaller.md/example-1-compiled.png)

Opening both the GNU/Linux executable and the OS X APP file yields the same result:

![example-1-output](/images/posts/2019-8-19-Bundle-PySimpleGui-scripts-as-OSX-APP-files-with-PyInstaller.md/example-1-output.png)

This means that our Python script was successfully packaged into an executable (including the imported modules): Hurray!

### Example 2 - Additional data

Before we continue, please delete the `build`, `dist` and `<program_name>.spec` that were generated, as these might interrupt the packaging process.

Now that we know how to package a simple script, let's cover the case where additional data or resources are required.

Again, let's take the simple "Hello World!" script as an example. This time we want our GUI to display an image of the earth and to also use it as an icon (**make sure it is in `.icns` format and of size 512x512 px**). We will place the `.png` in a `Resources` folder and we will keep our icon in the document root. Our file hierarchy therefor looks like this:

```bash
.
|____hello-world.py
|____icon.icns
|____Resources
| |____earth.png
|____package.sh
```

Now comes the tricky part: to access these resources you will need to define the following function in your script, which given the relative path, will return the absolute path to a file (I found this function on stack exchange somewhere, but I cant find the page anymore  >-<):

```python
def resource_path(relative_path):
    """ Get absolute path to resource, works for dev and for PyInstaller """
    try:
        # PyInstaller creates a temp folder and stores path in _MEIPASS
        base_path = sys._MEIPASS
    except Exception:
        base_path = os.environ.get("_MEIPASS2",os.path.abspath("."))

    return os.path.join(base_path, relative_path)
```

**Be sure to add the following import statements**

```python
# Required imports for resource_path()
import sys
import os
```

**Each time you define a path to a resource, you will need to pass the relative path to the resource_path() function.**

Therefor our `hello-world.py` scripts now looks something like this:

```python
import PySimpleGUI as sg # Import PySimpleGui module
import random # impot * statement

# Required imports for resource_path()
import sys
import os

# Define functions
def resource_path(relative_path):
    """ Get absolute path to resource, works for dev and for PyInstaller """
    try:
        # PyInstaller creates a temp folder and stores path in _MEIPASS
        base_path = sys._MEIPASS
    except Exception:
        base_path = os.environ.get("_MEIPASS2",os.path.abspath("."))

    return os.path.join(base_path, relative_path)

image_earth = resource_path("./Resources/earth.png") # Define earth image
image_icon = resource_path("icon.icns") # Define icon

layout= [[sg.Text("Hello World! " + str(random.randint(0, 10)), font=("Helvetica", 30)), sg.Image(filename=image_earth)]] # Define layout

window = sg.Window('Hello World!', layout, icon=image_icon)

while(True): # Start event loop
    event, values = window.Read()

    if event is None: # Quit
        window.Close()
        sys.exit(0)

```

Let's break it down:

* `image_earth` is the path to the earth image. Since `earth.png` is located in `./Resources` we will have to pass the relative path `./Resources/earth.png` to the `resource_path()` function. Thus `image_earth = resource_path("./Resources/earth.png")`. **This setup has to be implemented to all external resources**.
* `icon` is the path to the icon. Same set up as  `image_earth`.
* Be sure to pass the `icon` parametre **whenever possible** (e.g. when initializing an `sg.Window` or an `sg.Popup` objects) otherwise your icon will get messed up.

To package our `hello-world.py` script, the command wil thus be:

```bash
pyinstaller --onefile --windowed --add-binary='/System/Library/Frameworks/Tk.framework/Tk':'tk' --add-binary='/System/Library/Frameworks/Tcl.framework/Tcl':'tcl' -i='icon.icns' --add-data='./Resources':'/Resources' hello-world.py <<< y
```

Let's see what's going on:

* `-i='icon.icns'` specifies that the application icon is `icon.icns`. Note that if you move your icon inside a folder, you will have to specify the relative path to the icon (i.e. if  `icon.icns` is in folder `folder`, then the flag would be `-i='./folder/icon.icns'`)
* `--add-data='./Resources':'/Resources'` specifies that the folder `/Resources` is required to run the application. The syntax `'./Resources':'/Resources'` specifies folder source and destination. **You must pass this flag for every folder you want to include.** Note that additional data cannot be stored in the application document root (i.e. you cannot copy `./Resources` to `/`)

There is one last step to make our application work, and that is to **manually copy** the `./Resources` folder in the specified location (i.e `/Resources`). To do this we can use the `cp` command:

```bash
cp Resources/* dist/hello-world.app/Contents/Resources
```

I normally create a `package.sh` file where I concatenate the two commands.

Your APP file hierarchy should now look like this:

```bash
.
|____Contents
| |____MacOS
| | |____hello-world
| |____Resources
| | |____icon.icns
| | |____earth.png
| |____Frameworks
| |____Info.plist
```

Already we can see that our icon was applied to the APP file:

![example-2-compiled](/images/posts/2019-8-19-Bundle-PySimpleGui-scripts-as-OSX-APP-files-with-PyInstaller.md/examole-2-compiled.png)

And when we start the `hello-world.app` file we can see that our image was loaded successfully:

![example-2-output](/images/posts/2019-8-19-Bundle-PySimpleGui-scripts-as-OSX-APP-files-with-PyInstaller.md/example-2-output.png)

You can now rename `hello-world` to whatever you want and distribute it to your users: Hurray!

Thanks for reading :)

If you have any questions or comments be sure to post them in the section below.

If you want to contact me personally, you can add me as friend on discord `@Marcel_Ferrari#0967` :D
