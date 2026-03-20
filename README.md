# Mini Linux Container

I wrote this to figure out how Linux containers actually work under the hood. Instead of just using Docker or LXC, I wanted to see what it takes to sandbox a process from scratch using C. 

It's essentially a minimal container runtime that sets up an isolated environment for a process to run in.

## What's happening under the hood?

This relies on a few core Linux kernel features to build the sandbox:
* **Namespaces:** Isolates the process tree (PID), network, hostname (UTS), IPC, and mount points.
* **Cgroups (v1):** Sets hard limits on resources. It restricts memory usage, CPU time, and caps the maximum number of PIDs to prevent a fork bomb from taking down the host system.
* **Capabilities:** Even if the user is `root` inside the container, the code drops a massive list of dangerous kernel capabilities (like `CAP_SYS_BOOT`, `CAP_SYS_MODULE`, `CAP_MAC_ADMIN`, etc.).
* **Seccomp:** A custom system call filter to block specific syscalls that could be used to break out of the sandbox.

## Building

You'll need a Linux machine, GCC, and a couple of libraries (`libcap` and `libseccomp`).

If you're on Debian/Ubuntu, grab the dependencies first:
` ` `bash
sudo apt-get update
sudo apt-get install build-essential libcap-dev libseccomp-dev
` ` `

To compile the project, just run:
` ` `bash
gcc -Wall -Werror -lcap -lseccomp contained.c -o contained
` ` `

## Running it

You have to run the executable as `root` (via `sudo`) because setting up namespaces, cgroups, and mounting filesystems requires host-level privileges.

**Syntax:**
```bash
sudo ./contained -m <rootfs_dir> -u <uid> -c <command> [args...]
```

**Example:**
If you want to pop a shell inside the container, using the current directory (`.`) as the root filesystem, running as UID `0`:
` ` `bash
sudo ./contained -m . -u 0 -c /bin/sh
` ` `

If everything is set up correctly, you'll see it building the sandbox, pivoting the root, dropping privileges, and handing you the shell:
` ` `text
=> validating Linux version...
=> setting cgroups...memory...cpu...pids...blkio...done.
=> setting rlimit...done.
=> remounting everything with MS_PRIVATE...remounted.
=> making a temp directory and a bind mount there...done.
=> pivoting root...done.
=> unmounting /oldroot...done.
=> switching to uid 0 / gid 0...done.
=> dropping capabilities...bounding...inheritable...done.
=> filtering syscalls...done.
/ # whoami
root
` ` `

