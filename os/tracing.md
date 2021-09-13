# Tracing Tools

## Strace

Acts as a proxy between application (eg the `ls` command line utility) and system calls. Thus running strace comes with a conciderable performance cost (ie - not wise to use in production).

Good for seeing what system calls an application is making and how much time they're taking.

Strace is installed on most linux OS's by default so Google if your particular flavour doesn't have it

### Useful commands

Check syscalls for `ls -lath` command and output to file:

```bash
strace -o output.txt ls -lath
```

List number of syscalls, errors, percentage and actual time taken for command:

```bash
strace -c ls -lath
```

**Note:** A few amount of errors appears to be normal. An abnormally high amount would need further investigation.

Look at a specific syscall which had some errors for the command. In this example, looking at the `openat` syscall

```bash
strace -e openat ls -lath
```

List timestamps with the syscalls (same example as above):

```bash
strace -t -e openat ls -lath
```

using real time (from delta 0):

```bash
strace -r -e openat ls -lath
```

Run strace against a process that is already running:

```bash
strace -p PID_OF_PROCESS
```

