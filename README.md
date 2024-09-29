# decomp-permuter-agbcc

Automatically permutes C files to better match a target binary. The permuter has two modes of operation:
- Random: purely at random, introduce temporary variables for values, change types, put statements on the same line...
- Manual: test all combinations of user-specified variations, using macros like `PERM_GENERAL(a = b ? c : d;, if (b) a = c; else a = d;)` to try both specified alternatives.

The modes can also be combined, by using the `PERM_RANDOMIZE` macro.

[<img src="https://asciinema.org/a/232846.svg" height="300">](https://asciinema.org/a/232846)

This tool supports ARMv4T assembly compiled by agbcc. For MIPS, PowerPC, and other AArch32 assembly support; please see [the upstream project](https://github.com/simonlindholm/decomp-permuter).

## Prerequisites

You will need to install some prerequisites:

* For pip-managed environments: `python3 -m pip install pycparser toml Levenshtein`
* For externally-managed environments with apt: `sudo apt install python3-pycparser python3-toml python3-levenshtein`

For Python 3.6 or earlier, you will also need to install `dataclasses`.

Levenshtein is an optional dependency and may be omitted (the default diff algorithm is difflib).

## Setup

This setup will need to be performed once per project. It is assumed you are using git for version control. It is assumed that all code is either decompiled or disassembled into `.s` files; no code should exclusively be in `.o` files or included from the base ROM. It is acceptable for data to still be included from the base ROM.

The setup process is NOT plug-and-play. Parts of the setup process will need to be adapted to your project.

### Step 1: Create a new branch

*Beyond this point, do not interact with git unless instructed to do so.*

If you are setting up the permuter for everyone who works on the project, create a new branch from the default upstream branch, then continue to step 2.

If you are setting up the permuter for only yourself, switch to the branch that you wish to add permuter support to, then make a new commit. The setup process will partially destroy the project directory. As such, it is critical that you create a commit before continuing with the setup process. A later step in the setup process will instruct you to discard all changes made since this commit.

### Step 2: Create two working directories

Create two temporary working directories *outside* of the project directory. It is recommended that you create these directories as siblings of the project directory, or as subdirectories of a sibling directory. However, you may create them anywhere as long as you remember their locations, and they are not inside the project directory. Do not include a space in either directory's full path.

### Step 3: Split .s files

1. Move (not copy) a `.s` file from the project directory to the first working directory.
2. Split this `.s` file such that each function is in its own file, plus one file for the header. As an example, if each function in the `.s` file begins with `thumb_func_start`, then a command to split this `.s` file appropriately would be `csplit --digits=3 filename.s /thumb_func_start/ '{*}'`. This command will need to be adjusted if you have 1000 or more functions in the `.s` file, or if some functions start with alternative macros (like `arm_func_start` or `non_word_aligned_thumb_func_start`).
3. Rename the header file. If you used `csplit`, this is probably `mv xx000 header`.
4. Rename the function files so that each file is `name_of_function.s`. This may look something like `for filename in ./xx*; do mv $filename "$(head -1 $filename | cut -c 19-)".s; done`. Note that the `19` is *not* plug-and-play friendly; you will need to experiment to determine the correct number. Make sure that there is not a leading space in the output of the `head -1 file | cut -c 19-` portion.
5. Concatenate the header onto each function file. If you used `csplit`, this may look like `for filename in ./*.s; do cat header $filename | sponge $filename; done`.
6. Move (not copy) all single-function `.s` files from the first working directory to the second working directory. If the original `.s` file or the header file still exist, delete them at this time. At the end of this step, the first working directory should be empty, and the second working directory should only contain `.s` files with exactly one function.
7. Repeat for all `.s` files in the project directory that contain functions.

### Step 4: Compile .s files to .o files

1. Move all `.s` files from the second working directory to the appropriate location within the project directory.
2. Run the standard build command for your project (typically `make`). It is expected that this command will end with an error. If necessary, modify your build command/pipeline so that all build artifacts are kept.
3. Navigate to your project's build directory, and ensure that each `.s` file has a corresponding `.o` file in the build directory. If you have fewer `.o` files than expected, something went wrong. You will need to read the errors the build command output to determine the issue. If you see an "invalid offset, value too big" error or something similar, see the section below.
4. Move all `.o` files from the project directory to the first working directory. (Ignore `.o` files from sources other than the single-function `.s` files.)
5. Run the standard cleanup command for your project (typically `make clean`). If multiple levels of cleanup exist, it is recommended to use the most extreme version (i.e., prefer `make clean` to `make tidy`).

#### Invalid offset, value too big

You may see an error like this:

![image](https://github.com/user-attachments/assets/6dd50993-35ff-4a98-a7b6-1525252a1388)

This error indicates that you did not split functions correctly *during disassembly*. The specific cause is that a code section is declared as the start of a new function when it actually belongs to the previous function. This happens when a function is so large that a branch (be it a loop, conditional statement, or early return) cannot be fulfilled with a `b` instruction. In these cases, agbcc will elect to use a `bl` instruction instead of a `bx` instruction where possible. For example, take this function with an early "function call":

![image](https://github.com/user-attachments/assets/cca4198a-cecf-44a3-b7e6-6598264e56e8)

Looking at the end of that function, we see there is no return:

![image](https://github.com/user-attachments/assets/14a2d435-2d0e-4766-82f9-bacd835f0e9d)

Looking at the "next" function, the one that was "called", we see a function epilogue but no prologue:

![image](https://github.com/user-attachments/assets/a322953a-a93f-40d6-906d-762ca1a96ef0)

These two functions are actually one function, and the early "function call" was actually an early return.

While it is possible to manually intervene and merge these functions without restarting the setup process, doing so is outside the scope of this document. If you do not feel comfortable with manual intervention, perform the following steps:

1. Delete the contents of both working directories
2. Perform step 5 below, but do not continue to step 6
3. Merge the incorrectly split functions
4. Begin again from step 1 (commit the changes)

### Step 5: Discard repository changes

Discard all changes made since the commit made in step 1. If you are using GitHub Desktop, this can be accomplished by right-clicking on `{number} changed files`, then selecting "Discard all changes...".

![image](https://github.com/user-attachments/assets/daa1be97-95d5-4004-a56e-703b12db8a26)


### Step 6: Move .o files to project directory

Create a new subdirectory in your project directory. The recommended name is `expected_objs` or something similar, and it is recommended to put it in the root of the project directory, but you may ignore these recommendations. Move all `.o` files from the first working directory to this new subdirectory.

### Step 7: Create the permuter config file

Create a new file in the root of your project directory named `permuter_settings.toml`. In this file, add the following lines, replacing `$(CC)`, `$(CFLAGS)`, etc. with their values for your project:

```
compiler_type = "gcc"
compiler_command = "$(CC) $(CFLAGS) -o /dev/stdout | $(AS) $(ASFLAGS)"
assembler_command = "$(AS) $(ASFLAGS)"
```

For example, pokepinballrs's `permuter_settings.toml` looks like this:

```
compiler_type = "gcc"
compiler_command = "tools/agbcc/bin/agbcc -mthumb-interwork -Wimplicit -Wparentheses -Werror -O2 -fhex-asm -fprologue-bugfix -o /dev/stdout | arm-none-eabi-as -mcpu=arm7tdmi"
assembler_command = "arm-none-eabi-as -mcpu=arm7tdmi"
```

If you use ninja as your build system, you will need to add `build_system = "ninja"` to this file.

In most instances, decomp-permuter-agbcc will handle C preprocessor flags (`$(CPPFLAGS)`) for you. In the event that the incorrect preprocessor flags are used, you will need to edit `import.py` manually.

### Step 8: (Optional) Create the objdiff config file

At this point, adding objdiff support to your project is trivial. To add objdiff support, create a new file in the root of your project named `objdiff.json`. In this file, add the following lines, replacing the values of `"target_dir"` and `"base_dir"` to match your project's directory structure:

```
{
  "$schema": "https://raw.githubusercontent.com/encounter/objdiff/main/config.schema.json",
  "target_dir": "expected_objs",
  "build_target": false,
  "base_dir": "build/pokepinballrs/src",
  "build_base": true,
  "watch_patterns": [
    "*.c",
    "*.cp",
    "*.cpp",
    "*.cxx",
    "*.h",
    "*.hp",
    "*.hpp",
    "*.hxx",
    "*.s",
    "*.S",
    "*.asm",
    "*.inc",
    "*.py",
    "*.yml",
    "*.txt",
    "*.json"
  ]
}
```

### Step 9: Update .gitignore

If you are setting up the permuter for only yourself, skip this step.

Add the following lines to your `.gitignore` file, adjusting the path as necessary:

```
!expected_objs/*.o
!permuter_settings.toml
!objdiff.json
```

You should now see all the `.o` files from step 6 and the config files from steps 7 and 8 in `git status` or in your GUI client. Commit the changes, and publish the branch.

### Step 10: Cleanup

You may now delete both working directories.

You may now resume interacting with git as normal.

Setup is now complete!

## Usage

First, isolate the function you wish to run the permuter on. This function will need to be in its own .c file. You do not need to remove this function from its original file, but be advised that your build pipeline may fail unless you do so (but see below).

Next, run the import script: `./import.py path/to/file.c path/to/file.o func_name`. This will create a directory that the permuter will work on. You can also manually create the directory. The directory must contain:
  - a .c file with a single function,
  - a .o file to match,
  - a .sh file that compiles the .c file (it will be invoked as `./compile.sh input.c -o output.o`),
  - a .toml file specifying settings.

The minimal .toml file is:
```
func_name = "(insert function name here)"
compiler_type = "gcc"
```

Once the directory has been made, you may delete the .c file containing the isolated function from your project. (Do not delete it from the newly created directory!) This will allow your build pipeline to begin working again.

To run the permuter on the new directory, run `./permuter.py directory/`. Pass `-h` to see possible flags. `-j` is suggested (enables multi-threaded mode).

All of the possible randomizations are assigned a weight value that affects the frequency with which the randomization is chosen.
The default set of weights is specified in `default_weights.toml` and vary based on the targeted compiler.
These weights can be overridden by modifying `settings.toml` in the input directory.

The .c file may be modified with any of the following macros which affect manual permutation:

- `PERM_GENERAL(a, b, ...)` expands to any of `a`, `b`, ...
- `PERM_VAR(a, b)` sets the meta-variable `a` to `b`, `PERM_VAR(a)` expands to the meta-variable `a`.
- `PERM_RANDOMIZE(code)` expands to `code`, but allows randomization within that region. Multiple regions may be specified. A `PERM_RANDOMIZE` block is automatically added when there are no PERM macros.
- `PERM_FORCE_SAMELINE(code)` expands to `code`, but joined to a single line after round-tripping through the C parser library (which normally puts statements on separate lines). Can be useful for IDO where same-lineness affects codegen.
- `PERM_LINESWAP(lines)` expands to a permutation of the ordered set of non-whitespace lines (split by `\n`). Each line must contain zero or more complete C statements. (For incomplete statements use `PERM_LINESWAP_TEXT`, which is slower because it has to repeatedly parse C code.)
- `PERM_INT(lo, hi)` expands to an integer between `lo` and `hi` (which must be constants).
- `PERM_IGNORE(code)` expands to `code`, without passing it through the C parser library (pycparser)/randomizer. This can be used to avoid parse errors for non-standard C, e.g. `asm` blocks.
- `PERM_PRETEND(code)` expands to `code` for the purpose of the C parser/randomizer, but gets removed afterwards. This can be used together with `PERM_IGNORE` to enable the permuter to deal with input it isn't designed for (e.g. inline functions, C++, non-code).
- `PERM_ONCE([key,] code)` expands to either `code` or to nothing, such that each unique key gets expanded exactly once. `key` defaults to `code`. For example, `PERM_ONCE(a;) b; PERM_ONCE(a;)` expands to either `a; b;` or `b; a;`.

Arguments are split by a commas, exluding commas inside parenthesis. `(,)` is a special escape sequence that resolves to `,`. 

Nested macros are allowed, so e.g.
```
PERM_VAR(delayed, )
PERM_GENERAL(stmt;, PERM_VAR(delayed, stmt;))
...
PERM_VAR(delayed)
```
is an alternative way of writing `PERM_ONCE`.

If any multi-choice PERM macros are provided, automatic randomization will be disabled; to enable it you need to surround the function (or the relevant parts of it) with `PERM_RANDOMIZE`.

## permuter@home

decomp-permuter-agbcc does not provide support for permuter@home. While permuter@home functionality has not been removed, issues related to this functionality will not be addressed. In addition, the primary permuter network does not support agbcc, so you will need to find/create another network.

If you wish to use permuter@home, you will need to install `pynacl` as an additional prerequisite.

The permuter supports a distributed mode, where people can donate processor power to your permuter runs to speed them up.
To use this, pass `-J` when running `permuter.py` and follow the instructions.
(This can be combined with regular `-j` flags.)
You will need to be granted access by someone who is already connected to a permuter network.

permuter@home is only available for a limited number of compilers
(see [the list](https://github.com/decompals/pah-docker) for the main permuter network),
and currently does not work on native Windows (but WSL does work).

To allow others to use your computer for permuter runs, do the following:

- install Docker (used for sandboxing and to ensure a consistent environment)
- if on Linux, add yourself to the Docker group: `sudo usermod -aG docker $USER`
  or set up [rootless Docker](https://docs.docker.com/engine/security/rootless/)
- install required packages: `python3 -m pip install docker`
- open a terminal, and run `./pah.py run-server` to start the worker server.
  There are a few required arguments (e.g. how many cores to use), see `--help` for more details.

Anyone who is granted access to permuter@home can run a worker.

To set up a new permuter network, see [src/net/controller/README.md](./src/net/controller/README.md).

## FAQ

**What do the scores mean?** The scores are computed by taking diffs of objdump'd .o
files, and giving different penalties for lines that are the same/use the same
instruction/are reordered/don't match at all. 0 means the function matches fully.
Stack positions are ignored unless --stack-diffs is passed (but beware that the
permuter is currently quite bad at resolving stack differences). For more details,
see scorer.py. It's far from a perfect system, and should probably be tweaked to
look at e.g. the register diff graph.

**What sort of non-matchings are the permuter good at?** It's generally best towards
the end, when mostly regalloc changes remain. If there are reorderings or functional
changes, it's often easy to resolve those by hand, and neither the scorer nor the
randomizer tends to play well with them.

**Should I use this instead of trying to match code by hand?** No, but it can be a good
complement. PERM macros can be used to quickly test lots of variations of a function at
once, in cases where there are interactions between several parts of a function.
The randomization mode often finds lots of nonsensical changes that improve regalloc
"by accident"; it's up to you to pick out the ones that look sensible. If none do,
it can still be useful to know which parts of the function need to be changed to get the
code nearer to matching. Having made one of the improvements, and the function can then be
permuted again, to find further possible improvements.

## Helping out

There's tons of room for helping out with the permuter!
Many more randomization passes could be added, the scoring function is far from optimal,
the permuter could be made easier to use, etc. etc. The GitHub Issues list has some ideas.

Ideally, `mypy permuter.py` and `./run-tests.sh` should succeed with no errors, and files
formatted with `black`. To setup a pre-commit hook for black, run:
```
pip install pre-commit black
pre-commit install
```
PRs that skip this are still welcome, however.
