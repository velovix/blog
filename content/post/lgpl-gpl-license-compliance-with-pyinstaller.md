---
title: "LGPL and GPL License Compliance with PyInstaller"
date: 2018-08-16T19:57:12-07:00
draft: true
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

This post will take a look at strategies I have used to identify and either
externalize or eliminate LGPL and GPL dependencies in a proprietary PyInstaller
project.

Please note that I am in no way a lawyer and it's the responsibility of each
company or individual to do their due diligence when it comes to license
compliance.

# Identifying Your Python Dependencies

Chances are that your project has a `requirements.txt` or `setup.py` file that
specifies your Python dependencies. Looking at the licenses for these projects
is a good start, but make sure that you also consult the licenses of your
*transitive dependencies* as well. Transitive dependencies are the dependencies
of your dependencies, and PyInstaller will package those alongside your direct
dependencies.

You can use `pip` to inspect your dependency's dependencies using `pip show`.
In the following example, I'm taking a look at falcon's dependencies.

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
Location: /home/velovix/.virtualenvs/brainframe_v0.7.0/lib/python3.6/site-packages
Requires: six, python-mimeparse
Required-by: 
```

If we take a look at the "Requires" section of this output, we can see that
falcon depends on the "six" and "python-mimeparse" packages. We need to make
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
it always starts with the `_MEI` prefix. As long as you're not running
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
??? from here until ???END lines may have been inserted/deleted
while to sift through all these, but nobody said license compliance was easy.


# GPL Licensing

If your product is not GPL licensed, you will not be able to use a GPL licensed
library. Any code that is licensed with GPL must be removed from your executable
before deployment and you will not be able to link to an external GPL library,
either. It should be easy to avoid GPL Python libraries by simply removing any
libraries that use GPL code from your environment with `pip`. C/C++ libraries
are not always so straightforward.

Despite my best efforts to not rely on any GPL software, PyInstaller still
managed to find and include a couple GPL C/C++ libraries in my executable. To
solve this problem, I apply a filter in my `build.spec` to remove these files
before the executable finishes building. Let's take a look at how.

A simple `build.spec` might look something like this:

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

In the above example, we keep a list of known GPL libraries, and filter out the
`binaries` field of any libraries that look like them. The resulting filtered
list is fed to the `EXE` constructor.

You're going to want to make sure your executable still works without these
libraries. In my case, it seems that these libraries were never actually linked
to at runtime so I was able to avoid using them. Your mileage may vary.

# LGPL Licensing

The LGPL license is complicated and may or may not be an option for your
proprietary application. Some companies and individuals choose not to use LGPL
licensed code in their proprietary applications at all due to concerns with how
the license may be interpreted. Like every other legal decision, I will leave
this to the discretion of the reader.

If you use LGPL code in your product, you must allow users to swap out that LGPL
component with a modified version. The following is the "Combined Works" excerpt
from the LGPL 3.0 regarding this.

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


