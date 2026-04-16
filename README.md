# 📖 get_next_line

> *Reading a line from a file descriptor, one line at a time.*

A 42 School project implementing a function that reads from a file descriptor and returns one line per call — including the newline character — while efficiently managing a leftover buffer using static variables.

---

## 📚 Table of Contents

- [What is get\_next\_line?](#what-is-get_next_line)
- [How It Works](#how-it-works)
- [Project Structure](#project-structure)
- [Function Breakdown](#function-breakdown)
- [Memory Management](#memory-management)
- [Bonus Part — Multiple File Descriptors](#bonus-part--multiple-file-descriptors)
- [Compilation](#compilation)
- [Usage Example](#usage-example)
- [Edge Cases Handled](#edge-cases-handled)

---

## What is get_next_line?

`get_next_line` is a C function with the following prototype:

```c
char *get_next_line(int fd);
```

When called repeatedly (e.g., inside a loop), it reads the file pointed to by the file descriptor `fd` one line at a time. Each call returns:

- The next line read, **including the terminating `\n`** character (if present).
- `NULL` when there is nothing left to read, or if an error occurs.

The function works on regular files **and** standard input (`stdin`).

---

## How It Works

The core challenge of `get_next_line` is that a single `read()` call may read more bytes than one line contains. The excess bytes (everything after the `\n`) must be saved for the next call. This is solved with a **static variable** — `left_str` — which persists between function calls.

Here is the overall flow on each call:

```
get_next_line(fd)
      │
      ▼
ft_read_to_left_str()          ← Reads from fd into left_str until '\n' found or EOF
      │
      │  left_str now contains at least one full line (or everything until EOF)
      ▼
ft_get_line()                  ← Extracts exactly one line from left_str
      │
      ▼
ft_new_left_str()              ← Removes the extracted line from left_str (saves remainder)
      │
      ▼
Returns the line to the caller
```

### Step-by-step

1. **`ft_read_to_left_str(fd, left_str)`**  
   Allocates a `BUFFER_SIZE + 1` buffer, then calls `read()` in a loop, appending to `left_str` via `ft_strjoin`, until either a `\n` is found in `left_str` or `read()` returns `0` (EOF).

2. **`ft_get_line(left_str)`**  
   Scans `left_str` up to the first `\n` (inclusive) and allocates a new string containing exactly that line.

3. **`ft_new_left_str(left_str)`**  
   Removes the returned line from `left_str`, freeing the old string and returning only the remainder (everything after the `\n`), ready for the next call.

---

## Project Structure

```
get_next_line/
├── get_next_line.c            # Core logic — mandatory part
├── get_next_line.h            # Header — prototypes & BUFFER_SIZE define
├── get_next_line_utils.c      # Helper functions — mandatory part
├── get_next_line_bonus.c      # Core logic — bonus (multi-fd)
├── get_next_line_bonus.h      # Header — bonus part
└── get_next_line_utils_bonus.c# Helper functions — bonus part
```

---

## Function Breakdown

### `get_next_line.c`

#### `get_next_line(int fd)`

```c
char *get_next_line(int fd);
```

Entry point. Validates the file descriptor and `BUFFER_SIZE`. Calls `ft_read_to_left_str` to fill the static buffer, extracts one line with `ft_get_line`, then updates the remaining buffer with `ft_new_left_str`.

The key is the `static char *left_str` — it retains its value between calls, acting as the "memory" of what was already read but not yet returned.

#### `ft_read_to_left_str(int fd, char *left_str)`

```c
char *ft_read_to_left_str(int fd, char *left_str);
```

Reads from `fd` in chunks of `BUFFER_SIZE` bytes, joining each chunk onto `left_str` using `ft_strjoin`. Stops as soon as a newline character is found in `left_str`, or when `read()` signals EOF (`rd_bytes == 0`). Returns `NULL` on a read error.

---

### `get_next_line_utils.c`

#### `ft_strlen(char *s)`
Returns the length of `s`. Returns `0` if `s` is `NULL`.

#### `ft_strchr(char *s, int c)`
Scans `s` for the character `c`. Returns a pointer to the first occurrence, or `NULL`. If `c == '\0'`, returns a pointer to the null terminator. This is used to check whether `left_str` already contains a newline.

#### `ft_strjoin(char *left_str, char *buff)`
Concatenates `left_str` and `buff` into a newly allocated string. Frees `left_str` before returning, since a new allocation is made. If `left_str` is `NULL` (first call), it initialises it as an empty string `""`.

#### `ft_get_line(char *left_str)`
Allocates and returns a string containing one line from `left_str` — everything up to and including the first `\n`, or up to `\0` if no newline exists. Returns `NULL` if `left_str` is empty.

#### `ft_new_left_str(char *left_str)`
Calculates how many characters were consumed by the returned line, skips past the `\n`, and returns a new string containing only the remainder. Frees the old `left_str`. Returns `NULL` if the buffer is exhausted (no characters remain after the line).

---

## Memory Management

Every allocation is tracked and freed correctly:

| Allocation | Freed by |
|---|---|
| `buff` in `ft_read_to_left_str` | Always freed before returning |
| `left_str` inside `ft_strjoin` | Freed and replaced by new joined string |
| `left_str` at EOF | Freed inside `ft_new_left_str` |
| Returned `line` | **Caller's responsibility** to `free()` |

> ⚠️ The caller must always `free()` the string returned by `get_next_line`.

---

## Bonus Part — Multiple File Descriptors

The bonus version uses a static **array** of pointers instead of a single static pointer:

```c
static char *left_str[4096];
```

Each file descriptor is used as an index into this array. This means the function independently tracks the read position for up to 4096 simultaneous file descriptors, with no interference between them.

```c
left_str[fd] = ft_read_to_left_str(fd, left_str[fd]);
line = ft_get_line(left_str[fd]);
left_str[fd] = ft_new_left_str(left_str[fd]);
```

This approach satisfies both bonus requirements:
- Uses only **one static variable** (an array counts as one variable).
- Correctly handles **multiple file descriptors** alternating between calls.

---

## Compilation

Compile with a custom buffer size using the `-D BUFFER_SIZE=n` flag:

```bash
# Mandatory part
cc -Wall -Wextra -Werror -D BUFFER_SIZE=42 get_next_line.c get_next_line_utils.c -o gnl

# Bonus part
cc -Wall -Wextra -Werror -D BUFFER_SIZE=42 get_next_line_bonus.c get_next_line_utils_bonus.c -o gnl_bonus
```

The default `BUFFER_SIZE` (when the flag is omitted) is `1000`, as defined in the header.

---

## Usage Example

```c
#include <fcntl.h>
#include <stdio.h>
#include "get_next_line.h"

int main(void)
{
    int     fd;
    char    *line;

    fd = open("file.txt", O_RDONLY);
    if (fd < 0)
        return (1);
    while ((line = get_next_line(fd)) != NULL)
    {
        printf("%s", line);
        free(line);
    }
    close(fd);
    return (0);
}
```

---

## Edge Cases Handled

| Case | Behaviour |
|---|---|
| `fd < 0` | Returns `NULL` immediately |
| `BUFFER_SIZE <= 0` | Returns `NULL` immediately |
| `read()` error | Frees buffer, returns `NULL` |
| Empty file | `ft_get_line` returns `NULL` on empty `left_str` |
| File with no trailing `\n` | Last line returned without `\n`, next call returns `NULL` |
| Very large `BUFFER_SIZE` | Works — reads everything in one or few chunks |
| `BUFFER_SIZE = 1` | Works — reads one byte at a time |
| `NULL` `left_str` on first call | `ft_strjoin` initialises it to `""` |
| Multiple fds (bonus) | Each fd has its own slot in the static array |

---