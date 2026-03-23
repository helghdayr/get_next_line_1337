# get_next_line

> A function that reads one line at a time from a file descriptor — handling any buffer size, multiple file descriptors simultaneously (bonus), and persistent state between calls using a static variable.

---

## Table of Contents

- [Overview](#overview)
- [Project Structure](#project-structure)
- [Compilation](#compilation)
- [Function Prototype](#function-prototype)
- [How It Works](#how-it-works)
  - [The Core Problem](#the-core-problem)
  - [Static Variable — Persistent State](#static-variable--persistent-state)
  - [The Read Loop](#the-read-loop)
  - [Extracting One Line](#extracting-one-line)
  - [Updating the Leftover](#updating-the-leftover)
- [File by File](#file-by-file)
  - [get_next_line.h](#get_next_lineh)
  - [get_next_line.c](#get_next_linec)
  - [get_next_line_utils.c](#get_next_line_utilsc)
  - [get_next_line_bonus.h](#get_next_line_bonush)
  - [get_next_line_bonus.c](#get_next_line_bonusc)
  - [get_next_line_utils_bonus.c](#get_next_line_utils_bonusc)
- [BUFFER_SIZE](#buffer_size)
- [Key Concepts](#key-concepts)
- [Edge Cases](#edge-cases)
- [Mandatory Rules](#mandatory-rules)

---

## Overview

**get_next_line** is a 1337 / 42 School project. The goal is to write a function that, each time it is called, returns the **next line** from a file descriptor — including the trailing `'\n'` when present. Calling it repeatedly reads the file line by line until EOF, at which point it returns `NULL`.

The function must work correctly regardless of:
- How big or small `BUFFER_SIZE` is (even `1` or `10000000`)
- Whether the file ends with a newline or not
- Whether it is reading from a file, stdin, or a pipe

The **bonus** extends this to handle **multiple file descriptors at the same time** — for example, alternating calls between three open files without them interfering with each other.

---

## Project Structure

```
get_next_line/
├── get_next_line.h              # mandatory — prototype + BUFFER_SIZE define
├── get_next_line.c              # mandatory — main GNL logic (single fd)
├── get_next_line_utils.c        # mandatory — helper functions (strjoin, strchr, ...)
├── get_next_line_bonus.h        # bonus — prototype + BUFFER_SIZE define
├── get_next_line_bonus.c        # bonus — GNL with multiple fd support
└── get_next_line_utils_bonus.c  # bonus — helper functions for bonus version
```

> The mandatory and bonus versions are **separate sets of files** — they share the same logic but the bonus version adapts the static variable to hold state for multiple file descriptors at once.

---

## Compilation

```bash
# Compile mandatory version into your project
cc -Wall -Wextra -Werror -D BUFFER_SIZE=42 get_next_line.c get_next_line_utils.c -o program

# Compile bonus version (multiple fd support)
cc -Wall -Wextra -Werror -D BUFFER_SIZE=42 get_next_line_bonus.c get_next_line_utils_bonus.c -o program

# Override BUFFER_SIZE at compile time (any positive value)
cc -D BUFFER_SIZE=1    ...   # read one byte at a time
cc -D BUFFER_SIZE=8192 ...   # read 8KB chunks
```

`BUFFER_SIZE` can also be defined inside `get_next_line.h` as a fallback:

```c
#ifndef BUFFER_SIZE
# define BUFFER_SIZE 42
#endif
```

---

## Function Prototype

```c
char    *get_next_line(int fd);
```

| Parameter | Description |
|-----------|-------------|
| `fd` | File descriptor to read from (`0` = stdin, or any open file fd) |

| Return value | Condition |
|-------------|-----------|
| `"line content\n"` | A line was successfully read (includes `\n` if present) |
| `"last line"` | Last line of file with no trailing newline |
| `NULL` | EOF reached, or an error occurred |

---

## How It Works

### The Core Problem

Reading a file line by line sounds simple, but there is a fundamental mismatch: `read()` fills a fixed-size buffer of `BUFFER_SIZE` bytes — it has no concept of lines. A single `read()` call might:

- Return a **partial line** (the `'\n'` hasn't arrived yet)
- Return **multiple lines** (the buffer captured several `'\n'` characters)
- Return **exactly one line** (lucky edge case)

```
File content:  "Hello\nWorld\nFoo\n"
BUFFER_SIZE=8

read() call 1 → "Hello\nWo"   ← contains one full line AND start of next
read() call 2 → "rld\nFoo\n"  ← contains the rest
```

The solution: keep a **leftover** — a string that stores everything read so far that has not yet been returned as a line. Each call to `get_next_line` feeds new bytes from `read()` into this leftover until a `'\n'` is found, then carves out exactly one line.

---

### Static Variable — Persistent State

> **Concept:** A `static` local variable inside a function is initialized **once** and retains its value between calls. Unlike regular local variables, it is not destroyed when the function returns — it lives for the entire duration of the program.

```c
char    *get_next_line(int fd)
{
    static char *leftover;   // persists between calls — holds unread bytes
    // ...
}
```

On the **first call**, `leftover` is `NULL` (zero-initialized by the language).  
On **subsequent calls**, `leftover` contains the bytes that were read in a previous call but not yet returned as a line.

```
Call 1: leftover = NULL
        read() → "Hello\nWorld\n"
        leftover becomes "Hello\nWorld\n"
        extract line → return "Hello\n"
        leftover updated to "World\n"

Call 2: leftover = "World\n"
        '\n' already present — no read() needed
        extract line → return "World\n"
        leftover updated to ""  (or NULL)

Call 3: leftover = ""
        read() → "" (EOF, returns 0)
        nothing to return → return NULL
```

---

### The Read Loop

```c
char    buf[BUFFER_SIZE + 1];
int     bytes_read;

while (!ft_strchr(leftover, '\n'))
{
    bytes_read = read(fd, buf, BUFFER_SIZE);
    if (bytes_read <= 0)   // EOF or error
        break ;
    buf[bytes_read] = '\0';
    leftover = ft_strjoin(leftover, buf);  // append new bytes to leftover
}
```

The loop keeps reading **only as long as no `'\n'` has been accumulated**. Once a newline is in `leftover`, we stop reading and carve out the line. This is crucial — reading past the newline would consume bytes that belong to the next line.

---

### Extracting One Line

Once `leftover` contains at least one `'\n'` (or we've hit EOF), we extract the line:

```
leftover: "Hello\nWorld\n"
                ^
                find '\n' with ft_strchr

line:     "Hello\n"   ← everything up to and including '\n'
leftover: "World\n"   ← everything after '\n' (saved for next call)
```

```c
char    *ft_extract_line(char *leftover)
{
    int     i;
    char    *line;

    i = 0;
    while (leftover[i] && leftover[i] != '\n')
        i++;
    if (leftover[i] == '\n')
        i++;                          // include the '\n' in the line
    line = ft_substr(leftover, 0, i); // allocate exactly that many bytes
    return (line);
}
```

---

### Updating the Leftover

After extracting the line, the leftover must be updated to hold only the bytes that come **after** the extracted line:

```c
char    *ft_update_leftover(char *leftover)
{
    int     i;
    char    *new_leftover;

    i = 0;
    while (leftover[i] && leftover[i] != '\n')
        i++;
    if (!leftover[i])           // no '\n' found → nothing left (EOF case)
    {
        free(leftover);
        return (NULL);
    }
    new_leftover = ft_substr(leftover, i + 1, ft_strlen(leftover) - i - 1);
    free(leftover);
    return (new_leftover);
}
```

The old `leftover` is always `free`d before being replaced — forgetting this is the most common source of memory leaks in GNL.

---

## File by File

---

### `get_next_line.h`

**Role:** Header for the mandatory version.

Contains:
- The prototype of `get_next_line`
- The `BUFFER_SIZE` define (with `#ifndef` guard so it can be overridden at compile time)
- `#include` for `<unistd.h>` (`read`) and `<stdlib.h>` (`malloc`, `free`)
- Prototypes of utility functions from `get_next_line_utils.c`

```c
#ifndef GET_NEXT_LINE_H
# define GET_NEXT_LINE_H

# ifndef BUFFER_SIZE
#  define BUFFER_SIZE 42
# endif

# include <unistd.h>
# include <stdlib.h>

char    *get_next_line(int fd);
char    *ft_strjoin(char *s1, char *s2);
char    *ft_strchr(const char *s, int c);
size_t  ft_strlen(const char *s);
char    *ft_substr(char const *s, unsigned int start, size_t len);

#endif
```

---

### `get_next_line.c`

**Role:** The mandatory implementation — single file descriptor.

Contains the main `get_next_line` function:

1. Validates `fd` and `BUFFER_SIZE` (return `NULL` if invalid)
2. Holds a `static char *leftover` for the single fd
3. Runs the read loop until `'\n'` or EOF
4. Calls the extract function to get the current line
5. Calls the update function to trim the leftover
6. Returns the line (or `NULL` at EOF)

```c
char    *get_next_line(int fd)
{
    static char *leftover;
    char        *line;

    if (fd < 0 || BUFFER_SIZE <= 0)
        return (NULL);
    leftover = ft_read_and_join(fd, leftover);
    if (!leftover)
        return (NULL);
    line = ft_extract_line(leftover);
    leftover = ft_update_leftover(leftover);
    return (line);
}
```

---

### `get_next_line_utils.c`

**Role:** Helper functions for the mandatory version.

These functions support the main logic without cluttering `get_next_line.c`. They replicate the behavior of standard C functions — you cannot use `libc` string functions in this project.

| Function | Purpose |
|----------|---------|
| `ft_strlen` | Returns the length of a string (stop at `'\0'`) |
| `ft_strchr` | Returns pointer to first occurrence of a char in a string, or `NULL` |
| `ft_strjoin` | Allocates a new string = `s1 + s2`, then **frees `s1`** (the old leftover) |
| `ft_substr` | Returns a freshly allocated substring of `s` starting at `start`, length `len` |

> **Why does `ft_strjoin` free `s1`?**  
> In GNL, `s1` is always the old `leftover`. After joining, the old leftover is no longer needed. Having `ft_strjoin` free it internally prevents a pattern where every call site must `free` before reassigning — reducing the risk of leaks.

---

### `get_next_line_bonus.h`

**Role:** Header for the bonus version.

Identical structure to `get_next_line.h` but may define a different maximum fd count:

```c
# define MAX_FD 1024   // or use OPEN_MAX from <limits.h>
```

Prototypes of bonus utility functions are declared here.

---

### `get_next_line_bonus.c`

**Role:** Bonus implementation — supports **multiple file descriptors simultaneously**.

> **Concept:** The mandatory version uses a single `static char *leftover` — this works for one fd but breaks immediately when you alternate between two fds, because they'd share the same leftover buffer.
>
> The bonus solution: replace the single pointer with a **static array** indexed by `fd`. Each fd gets its own independent leftover slot.

```c
char    *get_next_line(int fd)
{
    static char *leftover[MAX_FD];   // one leftover slot per possible fd
    char        *line;

    if (fd < 0 || fd >= MAX_FD || BUFFER_SIZE <= 0)
        return (NULL);
    leftover[fd] = ft_read_and_join(fd, leftover[fd]);  // use slot [fd]
    if (!leftover[fd])
        return (NULL);
    line = ft_extract_line(leftover[fd]);
    leftover[fd] = ft_update_leftover(leftover[fd]);    // update slot [fd]
    return (line);
}
```

**Multiple fd example:**

```c
int fd1 = open("file1.txt", O_RDONLY);
int fd2 = open("file2.txt", O_RDONLY);
int fd3 = 0; // stdin

get_next_line(fd1); // reads from file1 — leftover[fd1] updated
get_next_line(fd2); // reads from file2 — leftover[fd2] updated, fd1 untouched
get_next_line(fd3); // reads from stdin — leftover[fd3] updated, fd1/fd2 untouched
get_next_line(fd1); // resumes file1 — leftover[fd1] still intact from first call
```

Each fd's state is fully isolated from all others.

---

### `get_next_line_utils_bonus.c`

**Role:** Helper functions for the bonus version.

Same functions as `get_next_line_utils.c` — `ft_strlen`, `ft_strchr`, `ft_strjoin`, `ft_substr`. They are duplicated into a separate file because the 42 norm requires bonus files to be entirely self-contained (the mandatory `.c` files cannot be included in the bonus compilation).

---

## BUFFER_SIZE

`BUFFER_SIZE` controls how many bytes `read()` requests from the OS in each call. It is a **compile-time constant** and does not affect correctness — only performance and the number of `read()` syscalls made.

| BUFFER_SIZE | Effect |
|-------------|--------|
| `1` | One byte per `read()` — correct but very slow (many syscalls) |
| `42` | Default — reasonable for testing |
| `8192` | Typical OS page size — efficient for large files |
| `10000000` | One giant read — works but allocates a huge stack buffer |

The function **must behave identically** for all valid values. This is the most important constraint of the project — it forces a design that never assumes anything about how much data one `read()` will return.

```bash
# Test with extreme values
cc -D BUFFER_SIZE=1        get_next_line.c get_next_line_utils.c -o gnl && ./gnl
cc -D BUFFER_SIZE=9999999  get_next_line.c get_next_line_utils.c -o gnl && ./gnl
```

---

## Key Concepts

### `read()` System Call

```c
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t count);
```

| Return value | Meaning |
|-------------|---------|
| `> 0` | Number of bytes actually read (may be less than `count`) |
| `0` | End of file — no more data |
| `-1` | Error — check `errno` |

`read()` does not null-terminate its buffer — you must do it manually: `buf[bytes_read] = '\0'`.

### Static Variables

A `static` local variable is:
- **Zero-initialized** on program start (pointers become `NULL`)
- **Persistent** — its value survives function returns
- **Private** — only accessible inside the function where it is declared

This makes it the perfect tool for "memory between calls" without needing global variables or passing extra parameters.

### Memory Management

Every `malloc` must eventually be paired with a `free`. In GNL the lifecycle is:

```
leftover starts as NULL
  │
  ├── ft_strjoin(leftover, buf)   → new malloc, old leftover freed inside strjoin
  ├── ft_extract_line(leftover)   → new malloc for the line (caller must free)
  └── ft_update_leftover(leftover)→ new malloc for remainder, old leftover freed
```

At EOF with no remaining data, `leftover` must be set to `NULL` — not just `free`d and left dangling. A dangling pointer that is read on the next call produces undefined behavior.

### File Descriptors

A file descriptor (`fd`) is a small non-negative integer the OS assigns to represent an open resource (file, pipe, socket, stdin). Standard descriptors:

| fd | Resource |
|----|----------|
| `0` | stdin |
| `1` | stdout |
| `2` | stderr |
| `3+` | Files opened with `open()` |

`get_next_line` works with any valid fd — it does not care whether the fd refers to a regular file, a pipe, or stdin.

---

## Edge Cases

| Scenario | Expected behavior |
|----------|------------------|
| `BUFFER_SIZE = 1` | Must work correctly — reads one byte per call |
| Empty file | First call returns `NULL` |
| File with no trailing `'\n'` | Last line is returned without `'\n'`, next call returns `NULL` |
| File with only `'\n'` | Returns `"\n"`, then `NULL` |
| Very long line (longer than `BUFFER_SIZE`) | Correctly accumulated across multiple reads |
| Multiple consecutive `'\n'` | Each empty line returned as `"\n"` |
| `fd = -1` | Returns `NULL` immediately |
| `BUFFER_SIZE <= 0` | Returns `NULL` immediately |
| `read()` returns `-1` (error) | Free leftover, return `NULL` — no crash |
| Bonus: alternating between fds | Each fd's state is completely isolated |
| Bonus: closing one fd mid-way | Other fds continue unaffected |

---

## Mandatory Rules

- [x] Only allowed functions: `read`, `malloc`, `free`
- [x] No use of `libft` (unless explicitly allowed by your school)
- [x] No global variables
- [x] `BUFFER_SIZE` must be overridable at compile time with `-D BUFFER_SIZE=N`
- [x] Function works for any `BUFFER_SIZE ≥ 1`
- [x] Returns the line including `'\n'` when present
- [x] Returns the last line even if it has no trailing `'\n'`
- [x] Returns `NULL` at EOF and on error
- [x] No memory leaks — every `malloc` is paired with a `free`
- [x] No undefined behavior — no dangling pointer reads after `free`
- [x] Bonus: single `get_next_line` function handles multiple fds via `static array`
- [x] Bonus files are fully self-contained (utils duplicated in `_bonus` files)
- [x] All files compile with `cc -Wall -Wextra -Werror`
- [x] Norminette compliant (1337 / 42 coding norm)

---

## Author

**Login:** `hael-ghd`  
**School:** 1337 / 42  
**Project:** get_next_line
