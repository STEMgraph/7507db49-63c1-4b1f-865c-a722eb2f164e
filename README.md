<!---
{
  "id": "7507db49-63c1-4b1f-865c-a722eb2f164e",
  "teaches": "GNU Make: Keeping the Source Tree Clean with a Build Directory",
  "depends_on": [
    "0432aac4-3185-48c6-a493-97b617687ecf"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-02",
  "keywords": [
    "make",
    "build directory",
    "out-of-tree build",
    "vpath",
    "$(dir)",
    "order-only prerequisite",
    "$(notdir)"
  ]
}
--->

# GNU Make: Keeping the Source Tree Clean with a Build Directory

> In this exercise you will move all build products out of the source directory into `build/`, teach `make` where to find sources with `vpath` and pattern rules across directories, and make `clean` a single `rm -rf`. No `sudo` is required.

## Introduction

Compiling into the source directory is convenient for a five-file project and a nuisance for anything larger: `ls` shows a mix of sources and objects, `git status` is full of noise, and switching between two configurations means recompiling everything. The standard answer is an **out-of-tree build**: sources live in `src/`, every generated file lives in `build/`, and `clean` deletes one directory.

With `make` this needs three ingredients:

- a pattern rule whose target and prerequisite live in different directories: `build/%.o: src/%.c`,
- a rule that creates the output directory before the first object is written, as an **order-only** prerequisite so that its timestamp never triggers rebuilds,
- optionally `vpath`, which tells `make` where to search for prerequisites it cannot find in the current directory.

### 1.1) Further Readings and Other Sources

* [GNU Make Manual: Searching Directories for Prerequisites (`VPATH`, `vpath`)](https://www.gnu.org/software/make/manual/html_node/Directory-Search.html)
* [GNU Make Manual: Types of Prerequisites](https://www.gnu.org/software/make/manual/html_node/Prerequisite-Types.html)
* [GNU Make Manual: Functions for File Names (`dir`, `notdir`, `addprefix`)](https://www.gnu.org/software/make/manual/html_node/File-Name-Functions.html)

## Tasks

### Task 1: Restructure the Project

```bash
cd ~/make-intro
mkdir -p src
git mv *.c *.h src/ 2>/dev/null || mv *.c *.h src/
rm -f *.o *.a *.d main_program
ls src
```

### Task 2: A Pattern Rule Across Directories

Rewrite the `Makefile`:

```make
CC      = gcc
CFLAGS  = -Wall -Wextra -O2 -MMD -MP
SRCDIR  = src
BUILD   = build

SRCS := $(wildcard $(SRCDIR)/*.c)
OBJS := $(patsubst $(SRCDIR)/%.c,$(BUILD)/%.o,$(SRCS))
DEPS := $(OBJS:.o=.d)

all: $(BUILD)/main_program

$(BUILD)/main_program: $(BUILD)/main.o $(BUILD)/libvec2.a $(BUILD)/libutil.a
	$(CC) $< -o $@ -L$(BUILD) -lvec2 -lutil -lm

$(BUILD)/libvec2.a: $(BUILD)/vec2.o
	ar rcs $@ $^

$(BUILD)/libutil.a: $(BUILD)/util.o
	ar rcs $@ $^

$(BUILD)/%.o: $(SRCDIR)/%.c | $(BUILD)
	$(CC) $(CFLAGS) -c $< -o $@

$(BUILD):
	mkdir -p $@

clean:
	rm -rf $(BUILD)

.PHONY: all clean
-include $(DEPS)
```

```bash
make
ls src build
./build/main_program
```

### Task 3: Why Order-Only?

Replace `| $(BUILD)` with ` $(BUILD)` (a normal prerequisite). Then:

```bash
make
make
touch build/libvec2.a
make
```

The last `make` recompiles objects that did not change, because writing `libvec2.a` changed the directory's timestamp. Restore the `|`.

### Task 4: Use `vpath` Instead of Spelling Out `src/`

An alternative that keeps the rules shorter: tell `make` where `.c` files live and write the pattern rule as if they were local.

```make
vpath %.c $(SRCDIR)
vpath %.h $(SRCDIR)

$(BUILD)/%.o: %.c | $(BUILD)
	$(CC) $(CFLAGS) -c $< -o $@
```

`$<` still expands to `src/main.c`, because `make` records where it found the file. Verify with `make clean && make -n`.

### Task 5: Nested Source Directories

Add a subdirectory and a file in it:

```bash
mkdir -p src/geom
printf '#include "../vec2.h"\ndouble GEOM_area(vec2 a, vec2 b) { return a.x * b.y - a.y * b.x; }\n' > src/geom/area.c
```

Change the source list and the directory rule so that the build tree mirrors the source tree:

```make
SRCS := $(shell find $(SRCDIR) -name '*.c')
OBJS := $(patsubst $(SRCDIR)/%.c,$(BUILD)/%.o,$(SRCS))

$(BUILD)/%.o: $(SRCDIR)/%.c
	@mkdir -p $(dir $@)
	$(CC) $(CFLAGS) -c $< -o $@
```

```bash
make clean && make
find build -name '*.o'
```

`$(dir $@)` is the directory part of the target. With `mkdir -p` inside the recipe the separate directory rule is no longer needed. Discuss which of the two approaches you prefer and why.

### Task 6: Two Build Directories Side by Side

```bash
make BUILD=build-debug CFLAGS="-g -O0 -MMD -MP"
make BUILD=build-release
ls -d build*
ls -l build-debug/main_program build-release/main_program
```

Both trees coexist and never invalidate each other. The next exercise turns this into proper configurations.

### Task 7: `.gitignore` and `clean`

```bash
printf 'build*/\n' > .gitignore
make clean
ls
```

The source tree is now exactly what you would commit.

## Questions

- In `$(BUILD)/%.o: $(SRCDIR)/%.c`, what does `%` match for the file `build/geom/area.o`?
- Why does a directory's timestamp change when a file inside it is created, but not when a file inside it is modified?
- What does `$<` expand to when `vpath` found the file in another directory? What about `$@`?
- Why is `$(shell find ...)` evaluated with `:=` and what would happen with `=`?
- `@mkdir -p $(dir $@)` runs on every compilation. Is that a correctness problem, a performance problem, or neither?
- CMake insists on `cmake -B build`. Which of the problems from this exercise does that avoid?

## Advice

Out-of-tree builds are not a `make` feature, they are a discipline: every generated file has a path that starts with `$(BUILD)`. Once you keep to it, `clean` is trivial, several configurations coexist, and your version control never sees an object file.

The order-only prerequisite for the directory is the one piece of `make` syntax from this exercise that people forget. If you ever see a project rebuilding everything after `make install`, a directory timestamp is the first suspect.
