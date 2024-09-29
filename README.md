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

If you are setting up the permuter for only yourself, switch to the branch that you wish to add permuter support to, then make a new commit. The setup process will partially destroy the project directory. As such, it is critical that you create a commit before continuing with the setup process. A later step in the setup process will instruct you to revert all changes made since this commit.

### Step 2: Create two working directories

Create two temporary working directories *outside* of the project directory. It is recommended that you create these directories as siblings of the project directory, but you may create them anywhere as long as you remember their locations, and they are not inside the project directory. Do not include a space in either directory's full path.

### Step 3: Split .s files

1. Move (not copy) a `.s` file from the project directory to the first working directory.
2. Split this `.s` file such that each function is in its own file, plus one file for the header. As an example, if each function in the `.s` file begins with `thumb_func_start`, then a command to split this `.s` file appropriately would be `csplit --digits=3 filename.s /thumb_func_start/ '{*}'`. This command will need to be adjusted if you have 1000 or more functions in the `.s` file, or if some functions start with alternative macros (like `arm_func_start` or `non_word_aligned_thumb_func_start`).
3. Concatenate the header onto each function file. If you used `csplit`, this may look like `mv xx000 header; for filename in ./xx*; do cat header $filename | sponge $filename; done`.
4. Rename the function files so that each file is `name_of_function.s`. This may look something like `for filename in ./xx*; do mv $filename "$(head -1 $filename | cut -c 19-)".s; done`. Note that the `19` is *not* plug-and-play friendly; you will need to experiment to determine the correct number. Make sure that there is not a leading space in the output of the `head -1 file | cut -c 19-` portion.
5. Move (not copy) all single-function `.s` files from the first working directory to the second working directory. If the original `.s` file or the header file still exist, delete them at this time. At the end of this step, the first working directory should be empty, and the second working directory should only contain `.s` files with exactly one function.
6. Repeat for all `.s` files in the project directory that contain functions.

### Step 4: Compile .s files to .o files

1. Move all `.s` files from the second working directory to the appropriate location within the project directory.
2. Run the standard build command for your project (typically `make`). It is expected that this command will end with an error. If necessary, modify your build command/pipeline so that all build artifacts are kept.
3. Navigate to your project's build directory, and ensure that each `.s` file has a corresponding `.o` file in the build directory. If you have less `.o` files than expected, see below for troubleshooting steps.
4. Move all `.o` files from the project directory to the first working directory. (Ignore `.o` files from sources other than the single-function `.s` files.)
5. Run the standard cleanup command for your project (typically `make clean`). If multiple levels of cleanup exist, it is recommended to use the most extreme version (i.e., prefer `make clean` to `make tidy`).

### Step 5: Revert to commit

Discard all changes made since the commit made in step 1. If you are using GitHub Desktop, this can be accomplished by right-clicking on `{number} changed files`, then selecting "Discard all changes...".

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

## Usage

`./permuter.py directory/` runs the permuter; see below for the meaning of the directory.
Pass `-h` to see possible flags. `-j` is suggested (enables multi-threaded mode).

You'll first need to install a couple of prerequisites: `python3 -m pip install pycparser pynacl toml Levenshtein` (also `dataclasses` if on Python 3.6 or below)
`pynacl` is optional and only necessary for the "permuter@home" networking feature.
`Levenshtein` is optional and only necessary for using the Levenshtein diff algorithm (difflib is used by default).

The permuter expects as input one or more directory containing:
  - a .c file with a single function,
  - a .o file to match,
  - a .sh file that compiles the .c file,
  - a .toml file specifying settings.

For projects with a properly configured makefile, you should be able to set these up by running
```
./import.py <path/to/file.c> <path/to/file.s>
```
where file.c contains the function to be permuted, and file.s is its assembly in a self-contained file.
Otherwise, see USAGE.md for more details.

For projects using Ninja instead of Make, add a `permuter_settings.toml` in the root or `tools/` directory of the project:
```toml
build_system = "ninja"
```
Then `import.py` should work as expected if `build.ninja` is at the root of the project.

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
