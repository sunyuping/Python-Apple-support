# Usage Guide

## The easy way

The easist way to use these packages is by creating a project with `Briefcase
<https://github.com/beeware/briefcase>`__. Briefcase will download pre-compiled
versions of these support packages, and add them to an Xcode project (or
pre-build stub application, in the case of macOS).

## The manual way

The Python support package *can* be manually added to any Xcode project;
however, you'll need to perform some steps manually (essentially reproducing what
Briefcase is doing)

**NOTE** Briefcase usage is the officially supported approach for using this
support package. If you are experiencing diffculties, one approach for debugging
is to generate a "Hello World" project with Briefcase, and compare the project that
Briefcase has generated with your own project.

To add this support package to your own project:

1. [Download a release tarball for your desired Python version and Apple platform](https://github.com/beeware/Python-Apple-support/releases)

2. Add the `python-stdlib` and `Python.xcframework` to your Xcode project. Both
   the `python-stdlib` folder and the `Python.xcframework` should be members of
   any target that needs to use Python.

3. In Xcode, select the root node of the project tree, and select the target you
   want to build.

4. Select "General" -> "Frameworks, Libraries and Embedded Content", and ensure
   that `Python.xcframework` is on the list of frameworks. It should be marked
   "Do not embed".

5. Select "General" -> "Build Phases", and ensure that the `python-stdlib` folder
   is listed in the "Copy Bundle Resources" step.

6. Add a new "Run script" build phase named "Sign Python Binary Modules", with the following content:

```bash
set -e
echo "Signing as $EXPANDED_CODE_SIGN_IDENTITY_NAME ($EXPANDED_CODE_SIGN_IDENTITY)"
find "$CODESIGNING_FOLDER_PATH/Contents/Resources/python-stdlib/lib-dynload" -name "*.so" -exec /usr/bin/codesign --force --sign "$EXPANDED_CODE_SIGN_IDENTITY" -o runtime --timestamp=none --preserve-metadata=identifier,entitlements,flags --generate-entitlement-der {} \;
```

You will now be able to access the Python runtime in your Python code.

If you are on iOS, you will be able to deploy to an iOS simulator without specifying
development team; however, you will need to specify a valid development team to sign
the binaries for deployment onto a physical device (or for submission to the App Store).

If you are on macOS, you will need to specify a valid development team to run
the app. If you don't want to specify a development team in your project, you
will also need to enable the "Disable Library Validation" entitlement under
"Signing & Capabilities" -> "Hardened Runtime" for your project.

## Accessing the Python runtime

There are 2 ways to access the Python runtime in your project code.

### Embedded C API.

You can use the [Python Embedded C
API](https://docs.python.org/3/extending/embedding.html) to instantiate a Python
interpreter. This is the approach taken by Briefcase; you may find the bootstrap
mainline code generated by Briefcase a helpful guide to what is needed to start
an interpreter and run Python code.

### PythonKit

An alternate approach is to use
[PythonKit](https://github.com/pvieito/PythonKit). PythonKit is a package that
provides a Swift API to running Python code.

To use PythonKit in your project:

1. Add PythonKit to your project using the Swift Package manager. See the
   PythonKit documentation for details.

2. Create a file called `module.modulemap` inside
   `Python.xcframework/macos-arm64_x86_64/Headers/`, containing the following
   code:
```
module Python {
    umbrella header "Python.h"
    export *
    link "Python"
}
```

3. In your Swift code, initialize the Python runtime. This should generally be
   done as early as possible in the application's lifecycle, but definitely
   needs to be done before you invoke Python code:
```swift
import Python

guard let stdLibPath = Bundle.main.path(forResource: "python-stdlib", ofType: nil) else { return }
guard let libDynloadPath = Bundle.main.path(forResource: "python-stdlib/lib-dynload", ofType: nil) else { return }
setenv("PYTHONHOME", stdLibPath, 1)
setenv("PYTHONPATH", "\(stdLibPath):\(libDynloadPath)", 1)
Py_Initialize()
// we now have a Python interpreter ready to be used
```

5. Invoke Python code in your app. For example:
```swift
import PythonKit

let sys = Python.import("sys")
print("Python Version: \(sys.version_info.major).\(sys.version_info.minor)")
print("Python Encoding: \(sys.getdefaultencoding().upper())")
print("Python Path: \(sys.path)")

_ = Python.import("math") // verifies `lib-dynload` is found and signed successfully
```

To integrate 3rd party python code and dependencies, you will need to make sure
`PYTHONPATH` contains their paths; once this has been done, you can run
`Python.import("<lib name>")`. to import that module from inside swift.
