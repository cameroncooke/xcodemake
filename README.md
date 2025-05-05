# xcodemake

[![Licence: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

`xcodemake` is a Perl script that converts the output of Apple's `xcodebuild` command into a standard `Makefile`. This allows developers to leverage the speed and simplicity of `make` for incremental builds of their Xcode projects, particularly beneficial when working outside the main Xcode IDE (e.g., in editors like VS Code, Cursor, Vim, Emacs).

## Why Use xcodemake?

*   **Faster Incremental Builds:** For changes only involving source code (`.swift`, `.m`, `.c`, `.cpp`), running `make` after initial generation is significantly faster than launching a full `xcodebuild` process, which involves considerable setup overhead.
*   **External Editor Integration:** Easily integrate your Xcode project builds into workflows based on external text editors and terminal environments.
*   **Simplicity:** Uses the well-understood `make` utility.
*   **Transparency:** The generated `Makefile` clearly shows the compilation and linking commands derived from your project's build log.

## Prerequisites

*   **macOS:** Required for Xcode.
*   **Xcode Command Line Tools:** Essential for `xcodebuild` and the compilers/linkers (`clang`, `swiftc`, `ld`, etc.). Install via `xcode-select --install` or ensure they are installed with Xcode.
*   **Perl:** Comes pre-installed on macOS. `xcodemake` is a Perl script. `JSON::PP` module is used, which is a core Perl module.

## Installation

1.  **Download/Clone:** Get the `xcodemake` script from this repository.
2.  **Make Executable:**
    ```bash
    chmod +x xcodemake
    ```
3.  **Place in PATH (Optional but Recommended):** Move or symlink the `xcodemake` script to a directory included in your shell's `$PATH` (e.g., `/usr/local/bin`, `~/bin`).

## Usage

The core idea is to run `xcodemake` *once* with the same arguments you would normally use to successfully build your project with `xcodebuild`. This captures the build process and generates the `Makefile`. After that, you use `make` for subsequent incremental builds.

### 1. First Run (Log Capture & Makefile Generation):

Navigate to your project's root directory (where the `.xcodeproj` file resides) in your terminal and run `xcodemake` followed by your `xcodebuild` arguments.

```bash
# Example:
xcodemake -project YourApp.xcodeproj -scheme YourApp -destination 'platform=iOS Simulator,name=iPhone 16,OS=latest' build
```

* **Important:** Provide all necessary arguments like -project, -workspace, -scheme, -sdk, -configuration, -destination, etc., just as you would for xcodebuild to perform a successful build.
* xcodemake will automatically add -config Debug if no -config or -configuration argument is found.
* **What happens:**
  * It runs xcodebuild ... clean first.
  * It then runs xcodebuild ... build, capturing both standard output and standard error.
  * The captured output is saved to a log file (e.g., xcodemake -project YourApp....log).
  * It parses this log file.
  * It generates a Makefile in the current directory based on the parsed build commands.

### 2. Subsequent Builds:
After the Makefile exists, simply run make in the same directory:
```bash
make
```

* make will check the modification times of your source files (.swift, .m, etc.) against their corresponding object files (.o).
* If a source file is newer, make will execute the compilation command found in the Makefile for that file.
* It will then execute the linking command if any object files were recompiled.
* Finally, it runs any codesign or touch commands found at the end of the original build process.

### 3. When is the Log Recaptured / Makefile Regenerated?
xcodemake is designed to automatically re-run the xcodebuild capture process and regenerate the Makefile under these conditions (checked *before* trying to run make):
* The xcodemake ...log file does not exist.
* The project file (YourProject.xcodeproj/project.pbxproj) has been modified more recently than the log file.
* The xcodemake script itself has been modified more recently than the Makefile.
* The specific xcodebuild path or arguments used (stored in the Makefile header) have changed since the Makefile was last generated.
* If make fails, xcodemake will attempt to run xcodebuild directly. If *that* succeeds, it implies project changes necessitated a full rebuild, so xcodemake will re-capture the log and regenerate the Makefile before trying make one last time.

## How it Works
1. **Log Capture:** Executes xcodebuild clean followed by xcodebuild build [args...], piping the combined output (stdout & stderr) to both the terminal (like tee) and a log file (xcodemake [args...].log). It avoids using env -i to ensure xcodebuild runs in a standard environment.
2. **Parsing:** Reads the generated log file line by line.
3. **Command Identification:** Looks for key lines indicating compilation or linking steps:
   * CompileC ...: For C, C++, Objective-C/C++ files.
   * SwiftDriver ... com.apple.xcode.tools.swift.compiler: The main driver command for Swift in newer Xcode versions.
   * SwiftCompile ...: Individual Swift file compilation (used in older Xcode versions, or as a fallback).
   * Ld ...: Linking steps for executables or dynamic libraries.
   * /usr/bin/codesign ...: Code signing steps.
   * /usr/bin/touch ...: Touching the final product.
4. **Rule Generation:**
   * For each compilation step (CompileC, SwiftDriver, SwiftCompile), it extracts the source file(s), the output object file(s) (.o), and the actual compiler command (clang or swiftc).
   * For SwiftDriver, it parses the -output-file-map JSON file specified in the command line to map source files to their corresponding object files. It then extracts the actual swiftc ... command by stripping the builtin-SwiftDriver prefix from the logged line.
   * It creates a standard Makefile rule:
     ```makefile
     # Example Swift rule (derived from SwiftDriver)
     /path/to/output/MyFile.o: /path/to/source/MyFile.swift
             cd /path/to/project
             /path/to/swiftc [LOTS OF ARGUMENTS...] && touch /path/to/output/MyFile.o
     ```
   * For linking (Ld), it identifies the output executable/dylib and the input object files (often listed in a -filelist). It creates a rule where the executable depends on the object files that xcodemake knows how to build (based on previous compilation rules).
   * codesign and touch commands found at the end of the log are added as recipes to the main build target (main).
5. **Timestamp Management:** The && touch ... command is appended to compilation recipes. This updates the object file's timestamp *only* if the compilation succeeds, ensuring make correctly tracks dependencies. The /usr/bin/time wrapper is *not* included in the final Makefile rules to ensure correct error propagation to make.

## Limitations
xcodemake provides significant speedups for source code iteration but has important limitations because it relies solely on parsing the *captured build log* rather than deeply analysing the project structure:
* **Source Files Only:** make will only automatically trigger recompilation if .swift, .m, .c, or .cpp files listed as dependencies in the Makefile are modified.
* **No Resource Tracking:** Changes to assets catalogs (.xcassets), storyboards (.storyboard), interface files (.xib), property lists (Info.plist), or other resources **will not** be detected by make.
* **No Project Setting Tracking:** Changes made within Xcode to build settings, phases, targets, or configurations in the project.pbxproj file **will not** be detected by make directly (though they *will* trigger an xcodemake re-capture on the next run because the .pbxproj file's modification time will change).
* **No Header Tracking (Generally):** While the *compiler* handles header dependencies during compilation, make itself won't typically see header files (.h, .hpp) as dependencies for object files unless they were explicitly listed in a way the script could parse (unlikely for standard C/Obj-C includes). Modifying *only* a header file might not trigger a recompile via make.
* **No Package Dependency Tracking:** Changes to Swift Package Manager (or Cocoapods/Carthage) dependencies are not tracked by make.
* **Fixed Build Environment:** The Makefile is generated based on the specific ARCHS, -destination, -configuration, etc., used during the initial xcodemake run. Building for a different architecture or configuration requires regenerating the Makefile by running xcodemake with the new arguments.

**Workaround for Limitations:** If you change anything other than the tracked source files (resources, project settings, headers, packages), you generally need to either:
1. Run the full xcodemake [args...] command again to recapture the build and regenerate the Makefile.
2. Run xcodebuild build [args...] directly.
3. Clean the build folder (make clean if you add a clean target, or manually) and then run make.

## Troubleshooting
* **make runs but nothing compiles after changes:** Ensure the files you changed are .swift, .m, etc., and are listed as dependencies in the Makefile. If you changed resources, project settings, or headers, run xcodemake ... again.
* **Errors during make:**
  * Check the compiler output for actual errors in your source code. make should now correctly report non-zero exit codes from the compiler.
  * If the errors seem related to project structure or missing files, the project might have changed significantly. Try running xcodemake ... again to regenerate the Makefile.
* **xcodemake fails during log capture:** Check the xcodebuild error output carefully. Ensure the arguments passed to xcodemake are correct and would result in a successful build if passed directly to xcodebuild.

## Licence
This project is licensed under the MIT Licence.