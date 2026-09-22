# Command Line Interpreter

A simplified command line interpreter (CLI) written in C++, implemented as the
course project for **Object Oriented Programming 1** at the School of Electrical
Engineering, University of Belgrade, academic year 2025/26.

The interpreter reads commands from its standard input one line at a time and
executes them. Commands can be entered interactively or supplied as a batch file.
Although far simpler than a real operating system shell, it implements many of the
same concepts: command arguments and options, input and output character streams,
stream redirection, pipelines, and structured error reporting.

---

## Status

Work in progress. The project is developed in two phases (see
[Project Phases](#project-phases)); Phase I is currently being implemented.

---

## Goal

The project is an exercise in **object oriented decomposition**, not merely in
getting a shell to work. The grading criteria of the course weigh the design at
least as heavily as the behaviour:

- clear abstractions with well defined responsibilities,
- high internal cohesion and loose external coupling between classes,
- modularity — clean separation into headers and translation units,
- encapsulation, sensible use of inheritance and class hierarchies,
- appropriate use of C++ language mechanisms (constructors, operator overloading,
  resource management, exceptions) *only where they are genuinely justified*,
- readability and ease of extension — the project is defended by extending or
  modifying it on the spot.

A procedural solution, regardless of whether it behaves correctly, is rejected.
The architecture is therefore designed so that Phase II can be attached to Phase I
without rewriting it.

---

## Behaviour

The interpreter runs as an interactive program. It repeatedly:

1. prints a command prompt (initially `$`) at the beginning of a new line,
2. reads one command line from standard input, terminated by `\n`,
3. executes the command (or commands) on that line,
4. repeats.

```
$ wc -w "Lorem ipsum dolor sit amet, consectetur adipiscing elit"
8
```

The general form of a single command is:

```
command [-opt] [argument]
```

where `command` is the command name, `-opt` is an option that affects how the
command executes, and `argument` is the command argument. The meaning of both
depends on the command itself.

---

## Command line syntax

### Lexical rules

- A command line is at most **512 `char` characters** long, not counting the
  terminating `\n`. If no newline is found within that limit, the remaining
  characters are discarded and the first 512 are taken as the command line.
- The interpreter is **case sensitive**.
- The command name, its option and its argument are separated by *whitespace* —
  sequences of spaces (`' '`) or horizontal tabs (`'\t'`).
- An argument enclosed in double quotes (`"`) is treated as a single integral
  character sequence and passed to the command without being split on whitespace.
  Inside quotes no character has any special meaning, except `\n` and `"` itself.
- The characters `|`, `<` and `>` have special meaning, unless they appear inside
  a quoted sequence, where they are ordinary characters.

If a character appears where the rules above do not permit it, the whole command
line is rejected and no command on it is executed. The error message reports the
position of the offending characters:

```
$ wc& -w *"Lorem ipsum dolor sit amet" +?
Error - unexpected characters:
wc& -w *"Lorem ipsum dolor sit amet" +?
  ^    ^                             ^^
```

### Input stream of a command

Most commands operate on characters taken from their *input character stream*,
which generalises four cases:

| Form | Input stream |
|---|---|
| `wc -w` | the console — characters typed by the user until `EOF` (`Ctrl+D` on Unix-like systems, `Ctrl+Z` on Windows) |
| `wc -w "Lorem ipsum"` | the quoted argument itself, without the quotes |
| `wc -w input.txt` | the contents of the named text file, relative to the current directory |
| `wc -w <input.txt` | the contents of the redirected file |

Input redirection with `<` is allowed **only** when the command does not already
have an argument that defines its input stream.

### Output stream of a command

Analogously, every command has an *output character stream*:

| Form | Output stream |
|---|---|
| `wc -w "..."` | the console |
| `wc -w "..." >out.txt` | the file `out.txt`; it is created if missing, and truncated if it already exists |
| `wc -w "..." >>out.txt` | the file `out.txt`, appended to; existing content is preserved |

Commands also have an *error output stream*, which is the console by default.

The redirection part may appear only at the end of a command. Whitespace around
`<`, `>` and `>>` is optional. Both streams of one command may be redirected, in
either order. A file name is a sequence of adjacent characters terminated by
whitespace, and is passed to the host operating system unchanged.

### Pipelines

A single command line may contain several commands connected into a *pipeline*
with `|`. The output stream of one command becomes the input stream of the next,
preserving character order:

```
$ time | tr -":" "." | wc -c > time.txt
```

Here `time` writes the current time, `tr` replaces every `:` with `.`, and `wc`
counts the characters of the transformed text and writes the result to `time.txt`.

Constraints:

- a command whose input or output stream is connected to a pipe may not define
  that same stream in any other way (as a literal argument, a file, or a
  redirection);
- commands that have no input stream (such as `time` and `date`) may appear only
  as the **first** command of a pipeline;
- commands that have no output stream may appear only as the **last** command.

---

## Command reference

| Command | Format | Description | Options |
|---|---|---|---|
| `echo` | `echo [argument]` | Copies characters from its input stream to its output stream unchanged. | — |
| `prompt` | `prompt argument` | Sets the command prompt to the quoted argument. | — |
| `time` | `time` | Writes the current system time to its output stream. | — |
| `date` | `date` | Writes the current system date to its output stream. | — |
| `touch` | `touch filename` | Creates an empty file in the current directory. Reports an error and does nothing else if the file already exists. | — |
| `truncate` | `truncate filename` | Deletes the contents of the named file. | — |
| `rm` | `rm filename` | Removes the named file from the file system. | — |
| `wc` | `wc -opt [argument]` | Counts words or characters read from its input stream and writes the count to its output stream. Words are sequences of characters separated by whitespace, as defined by `std::isspace`. | `-w` count words<br>`-c` count all characters |
| `tr` | `tr [argument] -what [with]` | Replaces every occurrence of the quoted sequence `what` with the quoted sequence `with` in the text read from the input stream, and writes the result to the output stream. If `with` is omitted, occurrences of `what` are removed. | — |
| `head` | `head -ncount [argument]` | Copies the first `count` lines of the input stream to the output stream and ignores the rest. | `-n` (mandatory) followed immediately by at most 5 decimal digits |
| `batch` | `batch filename` | Interprets the contents of the named file as a sequence of command lines separated by `\n`, exactly as if they had been read from the console. | — |

### Notes on `echo`

`echo` does not print its argument — it copies its *input stream*. The argument,
when present, is what defines that stream. Therefore `echo "text"` prints `text`,
`echo file.txt` prints the contents of `file.txt`, and a bare `echo` copies
console input until `EOF`.

### Notes on `batch`

`batch` processes command lines one by one, independently, until the end of the
input file. Within a batch:

- commands that were given no argument still read their input **from the console**,
  except those that are not the first command of a pipeline;
- the default output stream of the executed commands is the output stream of the
  `batch` command itself, although individual commands may redirect their own
  output;
- a command line that caused an error has its message written to the error output
  stream, and processing continues with the next line;
- the input file may contain any command, including `batch` itself. Recursion is
  not prevented; its consequences are the user's responsibility.

---

## Error handling

Every error encountered during lexical analysis, syntax analysis or execution
results in an error message and in abandoning that command **without any effect**.
The specification distinguishes six categories:

1. **Lexical errors** — illegal characters, reported with their position as shown
   above.
2. **Unknown command** — reported as `Unknown command: command`, together with the
   character sequence found where a command name was expected.
3. **Syntax errors** — in the format of a command or of the whole command line.
4. **Semantic errors in stream definition** — for example a command that has both
   an argument and an input redirection, or a command whose stream is connected to
   a pipe and also defined in another way.
5. **Semantic errors during execution** — where defined for a specific command.
6. **Errors reported by the operating system** — a file that does not exist where
   one is expected, insufficient access rights, and any other failure.

A lexical error invalidates the entire command line. Errors of the other
categories abandon only the command that caused them.

---

## Project phases

### Phase I — core

- Commands `echo`, `time`, `date`, `touch` and `wc`.
- Arguments and all options of those commands, including an input stream taken
  from a quoted argument or from a text file.
- Only error categories **5** and **6**. All test cases in this phase are
  lexically, syntactically and semantically valid.
- No input/output redirection and no pipelines.

### Phase II — full scope

Everything described in this document: the remaining commands (`prompt`,
`truncate`, `rm`, `tr`, `head`, `batch`), redirection, pipelines and the complete
error handling.

---

## Building and running

The project targets standard C++ and is built with the provided `Makefile`:

```sh
make            # build
make clean      # remove build artifacts
```

Run the interpreter interactively:

```sh
./cli
```

Or feed it a script through standard input:

```sh
./cli < tests/cases/001.in
```

The project must compile and link without errors. Compiler warnings are treated
as defects and resolved.

---

## Project structure

```
.
├── include/
│   ├── core/          # lexer, parser, streams, errors, interpreter, pipeline
│   └── commands/      # command hierarchy and command registry
├── src/
│   ├── core/          # implementation of include/core
│   ├── commands/      # implementation of include/commands
│   └── main.cpp       # entry point
├── tests/
│   ├── data/          # text files used as command input
│   ├── batch/         # scripts for the batch command
│   └── cases/         # test cases and their expected output
├── docs/              # design notes and documented assumptions
├── Makefile
└── README.md
```

Headers under `commands/` depend on `core/`, never the other way round, so the
directory layout mirrors the dependency layers of the design.

---

## Documented assumptions

Where the project specification leaves something undefined, the implementation
introduces a reasonable assumption and documents it in `docs/ASSUMPTIONS.md`, as
the specification requires. This covers, among other things, the exact set of
characters accepted inside an unquoted word and the behaviour in edge cases not
described by the specification.

---

## References

- [Course website](https://rti.etf.bg.ac.rs/rti/ir2oop/index.html) — Object
  Oriented Programming 1, School of Electrical Engineering, University of Belgrade
- [Lecture slides](https://rti.etf.bg.ac.rs/rti/ir2oo1/predavanja/)
