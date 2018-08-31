---
title: "LGPL and GPL License Compliance with PyInstaller"
date: 2018-08-31T19:57:12-07:00
draft: false
---

PyInstaller is a piece of software that attempts to bundle up a Python
application and all its dependencies into a single executable. It's an excellent
tool that makes deployments (especially to end users) much easier. It is capable
of bundling not only a project's Python dependencies, but its C/C++ library
dependencies as well.

This is great for any Python project that wants that self-contained,
single-executable experience that compiled languages can offer. It is also very
useful for proprietary projects that don't want to deploy their code directly
to a user. However, PyInstaller is a very aggressive packaging tool and pays no
mind to the licenses of the libraries it packages up. This poses a significant
challenge if your project does not make its source available, because you'll be
violating the licenses of any of your LGPL or GPL dependencies.

This post will take a look at strategies we have used to identify and either
externalize or eliminate LGPL and GPL dependencies in a proprietary PyInstaller
project.

Please note that I am in no way a lawyer and it's the responsibility of each
company or individual to do their due diligence when it comes to license
compliance.

# Identifying Your Python Dependencies

Chances are that your project has a `requirements.txt` or `setup.py` file that
specifies your Python dependencies. Looking at the licenses for these projects
is a good start, but make sure that you also consult the licenses of your
_transitive dependencies_ as well. Transitive dependencies are the dependencies
of your dependencies, and PyInstaller will package those alongside your direct
dependencies.

You can use `pip` to inspect your transitive dependencies using `pip show`.
In the following example, I'm taking a look at the dependencies of the `falcon`
package.

```
pip show falcon
```

This command outputs the following:

```
Name: falcon
Version: 1.4.1
Summary: An unladen web framework for building APIs and app backends.
Home-page: http://falconframework.org
Author: Kurt Griffiths
Author-email: mail@kgriffs.com
License: Apache 2.0
Location: /home/velovix/.virtualenvs/test/lib/python3.6/site-packages
Requires: six, python-mimeparse
Required-by:
```

If we take a look at the "Requires" section of this output, we can see that
falcon depends on the `six` and `python-mimeparse` packages. We need to make
sure we're adhering to the licenses of these packages as well.

# Identifying Your C/C++ Dependencies

Identifying C/C++ dependencies is less straightforward. It's not always
immediately clear what C/C++ libraries your project depends on and which ones
PyInstaller finds and bundles up for you. The best way I've found to know for
sure is to inspect the directory that a PyInstalled executable creates at
runtime.

When you run an executable made with PyInstaller, it starts by unpacking almost
everything that it is bundled with into a temporary directory. On Linux, this
directory is located in `/tmp`. The actual name of this directory is random but
it always starts with the `_MEI` prefix. As long as you're not running multiple
PyInstaller executables, this should be enough information to find the directory
in question.

Start by running your program like normal.

```
./MyCoolExecutable
```

While it's running, simply inspect the temporary directory that PyInstaller has
created. We're looking for `.so` files, which are shared library files
containing compiled C/C++ code. Using the names of these files, it is usually
relatively straightforward to find their corresponding project and license.

Depending on the size of your project, you may find quite a few shared library
files in this directory. My project contained more than 50. It may take quite a
while to sift through all these, but nobody said license compliance was easy.

When looking through these libraries, take note of the names of library files
that are GPL or LGPL licensed.

# GPL Licensing

If your product is not using a GPL-compatible license, you will not be able to
use a GPL licensed library. Any code that is licensed with GPL must be removed
from your executable before deployment and you will not be able to link to an
external GPL library, either. It should be easy to avoid GPL Python libraries
by simply removing any libraries that use GPL code from your environment with
`pip`. C/C++ libraries are not always so straightforward.

Despite our best efforts to not rely on any GPL software, PyInstaller still
managed to find and include a couple GPL C/C++ libraries in the executable. To
solve this problem, we apply a filter in our `build.spec` to remove these files
before the executable finishes building. Let's take a look at how.

A simple `build.spec` might look something like this:

```python
# -*- mode: python -*-

analysis = Analysis(['./my_main_file.py'],
    binaries=[],
    datas=[],
    hiddenimports=[],
    hookspath=[],
    runtime_hooks=[],
    excludes=[],
    win_no_prefer_redirects=False,
    win_private_assemblies=False,
    cipher=None)

pyz = PYZ(analysis.pure, analysis.zipped_data, cipher=None)

elf = EXE(pyz,
    analysis.scripts,
    analysis.binaries,
    analysis.zipfiles,
    analysis.datas,
    name='MyCoolExecutable',
    debug=False,
    strip=False,
    upx=True,
    console=True)
```

When you create a new `Analysis` object, PyInstaller does the work of
identifying your dependencies. The `binaries` field is a list of tuples in the
format `(filename, full_path, type)`. These fields represent all of the shared
object files that PyInstaller will put in your resulting executable. We may
simply filter out the libraries we don't want to include from this list. The
result might look something like this.

```
# -*- mode: python -*-

analysis = Analysis(['./my_main_file.py'],
    binaries=[],
    datas=[],
    hiddenimports=[],
    hookspath=[],
    runtime_hooks=[],
    excludes=[],
    win_no_prefer_redirects=False,
    win_private_assemblies=False,
    cipher=None)

gpl_libraries = ["libgcrypt.so", "libsystemd.so"]

def is_gpl(filename):
    for gpl_lib in gpl_libraries:
        if gpl_lib in filename:
            return True

    return False

filtered_binaries = []
for binary in analysis.binaries:
    filename, path, type_ = binary

    if is_gpl(filename):
        print("Excluding library:", filename)
    else:
        filtered_binaries.append(binary)


pyz = PYZ(analysis.pure, analysis.zipped_data, cipher=None)

elf = EXE(pyz,
    analysis.scripts,
    filtered_binaries,
    analysis.zipfiles,
    analysis.datas,
    name='MyCoolExecutable',
    debug=False,
    strip=False,
    upx=True,
    console=True)
```

In the above example, we keep a list of known GPL libraries and filter out the
`binaries` field of any libraries that look like them. The resulting filtered
list is fed to the `EXE` constructor.

You're going to want to make sure your executable still works without these
libraries. In my case, it seems that these libraries were never actually linked
to at runtime so we were able to avoid using them. Your mileage may vary.

# LGPL Licensing

The LGPL license is complicated and may or may not be an option for your
proprietary application. Some companies and individuals choose not to use LGPL
licensed code in their proprietary applications at all due to concerns with how
the license may be interpreted. Like every other legal decision, I will leave
this to the discretion of the reader.

If you decide to use LGPL code in your product, you must allow users to swap
out that LGPL component with a modified version. The following is the "Combined
Works" excerpt from the LGPL 3.0 regarding this.

```
  4. Combined Works.

  You may convey a Combined Work under terms of your choice that,
taken together, effectively do not restrict modification of the
portions of the Library contained in the Combined Work and reverse
engineering for debugging such modifications, if you also do each of
the following:

   a) Give prominent notice with each copy of the Combined Work that
   the Library is used in it and that the Library and its use are
   covered by this License.

   b) Accompany the Combined Work with a copy of the GNU GPL and this license
   document.

   c) For a Combined Work that displays copyright notices during
   execution, include the copyright notice for the Library among
   these notices, as well as a reference directing the user to the
   copies of the GNU GPL and this license document.

   d) Do one of the following:

       0) Convey the Minimal Corresponding Source under the terms of this
       License, and the Corresponding Application Code in a form
       suitable for, and under terms that permit, the user to
       recombine or relink the Application with a modified version of
       the Linked Version to produce a modified Combined Work, in the
       manner specified by section 6 of the GNU GPL for conveying
       Corresponding Source.

       1) Use a suitable shared library mechanism for linking with the
       Library.  A suitable mechanism is one that (a) uses at run time
       a copy of the Library already present on the user's computer
       system, and (b) will operate properly with a modified version
       of the Library that is interface-compatible with the Linked
       Version.

   e) Provide Installation Information, but only if you would otherwise
   be required to provide such information under section 6 of the
   GNU GPL, and only to the extent that such information is
   necessary to install and execute a modified version of the
   Combined Work produced by recombining or relinking the
   Application with a modified version of the Linked Version. (If
   you use option 4d0, the Installation Information must accompany
   the Minimal Corresponding Source and Corresponding Application
   Code. If you use option 4d1, you must provide the Installation
   Information in the manner specified by section 6 of the GNU GPL
   for conveying Corresponding Source.)
```

If your project distributes its source to the customer, complying with this
aspect of LGPL is straightforward. Simply include with your distribution
instructions on how to rebuild the executable with PyInstaller. Then the user
may install their own modified versions of any LGPL dependency and PyInstaller
will create an executable with that version instead. For closed-source
projects, things get more complicated.

## C/C++ Dependencies

Our goal in this section is to filter LGPL C/C++ libraries out of the
executable and put them into a separate `lib` folder. This folder will be
deployed alongside the executable. End users who wish to swap out the program's
LGPL dependencies with their own modified version may simply compile the
modified version into a shared library file and replace the one included in the
deployment. We can accomplish this in much the same way we did when excluding
GPL dependencies. Let's see how that might look.

```
# -*- mode: python -*-
from shutil import copyfile
from pathlib import Path

analysis = Analysis(['./my_main_file.py'],
    binaries=[],
    datas=[],
    hiddenimports=[],
    hookspath=[],
    runtime_hooks=[],
    excludes=[],
    win_no_prefer_redirects=False,
    win_private_assemblies=False,
    cipher=None)

gpl_libraries = ["libgcrypt.so", "libsystemd.so"]
lgpl_libraries = ["libavcodec", "libavformat"]

def is_gpl(filename):
    for gpl_lib in gpl_libraries:
        if gpl_lib in filename:
            return True

    return False

def is_lgpl(filename):
    for lgpl_lib in lgpl_libraries:
        if lgpl_lib in filename:
            return True

    return False

filtered_binaries = []
for binary in analysis.binaries:
    filename, path, type_ = binary

    if is_gpl(filename):
        print("Excluding library:", filename)
    elif is_lgpl(filename):
        print("Externalizing library:", filename")
        copyfile(path, Path("lib", filename))
    else:
        filtered_binaries.append(binary)


pyz = PYZ(analysis.pure, analysis.zipped_data, cipher=None)

elf = EXE(pyz,
    analysis.scripts,
    filtered_binaries,
    analysis.zipfiles,
    analysis.datas,
    name='MyCoolExecutable',
    debug=False,
    strip=False,
    upx=True,
    console=True)
```

Much like last time, we have a list of known LGPL dependencies and we attempt
to match them against each of our discovered library files. If there is a
match, we copy the file to a `lib` directory and exclude it from the bundling
process.

Now that these library files are no longer managed by PyInstaller, we need to
make sure the linker can find these library files at runtime. There are a few
ways to accomplish this, but we opted to deploy a script alongside the
executable that modifies the `LD_LIBRARY_PATH` variable to include the `lib`
directory.

```
#!/usr/bin/env bash

LD_LIBRARY_PATH=./lib:$LD_LIBRARY_PATH ./MyCoolExecutable
```

## Python Dependencies

PyInstaller does a lot in the background to make its Python dependency
management feel as seamless as it does. We need to take a moment to understand
how Python looks for modules and what PyInstaller does to change the
interpreter's default behavior in this respect before going forward and making
modifications of our own.

Note that these explanations are probably an oversimplification of what's
actually going on behind the scenes. I'm not an expert on this topic.

### The `sys.path` Variable

When you import a module, the interpreter looks in a variety of places to find
the corresponding Python file. It consults a list of directories one-by-one
until it finds a matching module. That list of directories is `sys.path`. The
`sys.path` variable is very similar in spirit to the `LD_LIBRARY_PATH`
environment variable that C/C++ developers are all too familiar with. It's also
a bit like the regular old `PATH` variable from Bash. They're all simply lists
of paths that instruct a program on where to look for something.

By default, the first entry in `sys.path` points to the same directory as the
Python script that you're running. The other entires depend on your
installation and environment. One of the entires will point to the location of
the standard library, for instance. If you're running in a virtual environment,
there will be entries pointing to where in that virtual environment your
dependences are stored. Basically, if you have a Python package installed
somewhere and you are able to import it, it's likely because there's a
`sys.path` entry pointing to that location.

For more information on `sys.path`, take a look at the
[official documentation][sys.path docs].

Keep in mind though that everything I've mentioned so far are just the
interpreter's defaults. Like so many other things in Python, this system can be
completely customized. This brings us to our next topic.

### The `sys.meta_path` Variable

For simple cases, editing `sys.path` is usually all you need to do to allow
Python to find your dependencies, but if you want to get into the nitty gritty
and completely change how Python finds modules, `sys.meta_path` is the tool to
reach for.

The `sys.meta_path` variable is a list of "meta path finders". A meta path
finder is an object that can find a module from the information in an import
statement. These objects expose a few key methods that the interpreter calls to
access this functionality. The built-in meta path finders consult `sys.path` to
do this work, but a custom meta path finder could use any method it likes in
order to find modules.

PyInstaller injects its own custom objects into `sys.meta_path` so that the
modules embedded in the executable can be found by the interpreter. These
meta path finders do not consult `sys.path`. If we want to add our own custom
module location, editing `sys.path` will not be enough. Instead, we need to
create a meta path finder of our own and inject it into `sys.meta_path`!

By the way, if you want to learn more about how PyInstaller does what it does,
take a look at their excellent [documentation][pyinstaller bootstrapping docs].

### The Solution

With all this background in mind, let's circle back to the original problem and
demonstrate how our newfound knowledge will help us solve it.

Say you're creating a proprietary application that depends on `python-vlc`.
This package is LGPL licensed, so we have to give users a way to substitute in
their own modified version. To do this, we're going to allow the user to set an
optional environment variable called `PYTHON_VLC_PATH`, whose value is a path
leading to the user's custom `python-vlc` implementation. If this environment
variable is set, we will import from that location instead of our prepackaged
version.

As we mentioned earlier, we're going to have to write a custom meta path finder
to accomplish this. The only required method for a meta path finder to
implement is `find_spec`. With this in mind, let's create a meta path finder
that can import a module given its name and the path to its implementation.

```python
class LGPLFinder:
    def __init__(self, module_name, custom_location):
        self.module_name = module_name
        self.custom_location = custom_location

    def find_spec(self, fullname, path, target=None):
        if fullname == self.module_name:
            source = os.path.join(self.custom_location,
                                  self.module_name + ".py")
            loader = SourceFileLoader(self.module_name, source)
            return ModuleSpec(self.module_name, loader)
        else:
            # Some other module, let other importers handle it
            return None
```

The class is fairly simple. You can see that we only try to import the module
if its `fullname` is equal to the module we're tasked with importing.
Otherwise, we return `None`, which tells the interpreter to consult the next
meta path finder in `sys.meta_path`. If it is our module of interest, we create
a path pointing to a Python file with the same name as the module in the custom
location provided earlier. We put this and the module's name into a
`SourceFileLoader`, a class that handles the details of how to load a Python
file into a module. What we return is a ModuleSpec, a class that describes
import-related information about a module.

I would definitely recommend reading up on these classes if you're curious
about the details. See the documentation for
[SourceFileLoader][SourceFileLoader docs] and [ModuleSpec][ModuleSpec docs].

Now that we have this custom meta path finder, let's take a look at how we
would inject it into `sys.meta_path`.

```python
python_vlc_path = os.environ.get("PYTHON_VLC_PATH", None)
if python_vlc_path is not None:
    finder = LGPLFinder("vlc", python_vlc_path)
    sys.meta_path.insert(0, finder)

    import vlc

    sys.meta_path.pop(0)
```

Once again, the code is surprisingly simple. We check to see if the user has
set the environment variable. If they have, we create a new LGPLFinder for
importing `python-vlc`. Before importing, we insert our finder at the very
beginning of `sys.meta_path`, ensuring it is the first meta path finder that
gets run. Then, we import `python-vlc` and remove the meta path finder from
`sys.meta_path`, since we no longer need it.

One important thing to know is that this code _must be run before your LGPL
dependency is imported anywhere else_. This is because modules in Python are
singletons. No matter how many times a module is imported in a project, the
initialization of the model only happens once and we need to make sure it
happens while our custom meta path loader is active. The benefit to this is
that your LGPL dependency can be imported normally everywhere else in your
project after this initial import.

# Conclusion

Achieving full license compliance is difficult and PyInstaller doesn't make it
any easier. However, we've demonstrated that it's still possible with enough
diligence and some knowledge about PyInstaller's inner workings.

[sys.path docs]: https://docs.python.org/3/library/sys.html#sys.path
[pyinstaller bootstrapping docs]: https://pyinstaller.readthedocs.io/en/v3.3.1/advanced-topics.html#the-bootstrap-process-in-detail
[SourceFileLoader docs]: https://docs.python.org/3/library/importlib.html#importlib.machinery.SourceFileLoader
[ModuleSpec docs]: https://docs.python.org/3/library/importlib.html#importlib.machinery.ModuleSpec
