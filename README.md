*This project has been created as part of the 42 curriculum by rmanzana*

# pipex

## Description

**pipex** reproduces the behaviour of a shell pipeline with redirections. This command:

```bash
./pipex infile "cmd1" "cmd2" outfile
```

behaves like:

```bash
< infile cmd1 | cmd2 > outfile
```

The project is an introduction to UNIX processes and inter-process communication: creating processes with `fork()`, connecting them with `pipe()`, redirecting standard input and output with `dup2()`, and running programs with `execve()`.

## How it works

### 1. Parsing

The four arguments are stored in a `t_command` structure: the input file, the output file, and each command split into an `argv` array (`"ls -l"` → `{"ls", "-l", NULL}`). Empty commands are rejected.

### 2. Finding the executables

`execve()` needs the full path of the program. If the command already starts with `/`, it is used directly. Otherwise, the `PATH` environment variable is split by `:` and each directory is tried (`/usr/bin/ls`, `/bin/ls`, …) until `access()` finds the file.

### 3. Execution

```
            pipe
 infile ──► cmd1 ──► [ write end | read end ] ──► cmd2 ──► outfile
          (child 1)                             (child 2)
```

1. A pipe is created with `pipe()`.
2. **First child:** opens `infile`, redirects it to `stdin` with `dup2()`, redirects `stdout` to the write end of the pipe, closes the unused descriptors and calls `execve()` with `cmd1`.
3. **Second child:** opens (or creates and truncates, with permissions `0644`) `outfile`, redirects the read end of the pipe to `stdin` and `stdout` to the file, then calls `execve()` with `cmd2`.
4. The parent closes both ends of the pipe and waits for its children with `waitpid()`, then returns an exit status based on theirs.

Closing every unused end of the pipe in each process matters: if a write end stays open somewhere, `cmd2` never receives `EOF` and the program hangs.

### 4. Error handling

Errors are reported on `stderr` with messages similar to the shell's: missing input file (`No such file or directory`), unknown command (`Command not found`, exit code `127`), permission problems, and failures of `pipe()`, `fork()` or `dup2()`. All allocated memory is freed on every path.

## Instructions

### Cloning

[`libft`](https://github.com/petandk/libft) is included as a **git submodule**, so clone with:

```bash
git clone --recurse-submodules https://github.com/petandk/pipex.git
# or, if already cloned:
git submodule update --init
```

### Compilation

```bash
make          # builds libft, ft_printf and the pipex executable
make clean    # removes object files
make fclean   # removes object files and the executable
make re       # rebuilds everything
```

The project uses its own `libft` and `ft_printf`, which are compiled automatically by the Makefile.

### Usage

```bash
./pipex infile "ls -l" "wc -l" outfile
# same as: < infile ls -l | wc -l > outfile

./pipex infile "grep a1" "wc -w" outfile
# same as: < infile grep a1 | wc -w > outfile
```

A sample `infile` is included in the repository.

## Resources

- [`pipe(2)`](https://man7.org/linux/man-pages/man2/pipe.2.html), [`fork(2)`](https://man7.org/linux/man-pages/man2/fork.2.html), [`dup2(2)`](https://man7.org/linux/man-pages/man2/dup.2.html), [`execve(2)`](https://man7.org/linux/man-pages/man2/execve.2.html) and [`waitpid(2)`](https://man7.org/linux/man-pages/man2/waitpid.2.html) manuals
- [Unix Processes in C — CodeVault (YouTube playlist)](https://www.youtube.com/playlist?list=PLfqABt5AS4FkW5mOn2Tn9ZZLLDwA3kZUY)
- [Pipelines — The Linux Programming Interface, chapter 44](https://man7.org/tlpi/)
- [Bash Reference Manual — Pipelines and Redirections](https://www.gnu.org/software/bash/manual/bash.html#Pipelines)
