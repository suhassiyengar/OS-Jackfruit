
OS-Jackfruit: Lightweight Container Runtime Report

Team Information
Name: Suhas S Iyengar,Sholka
SRN: PES1UG24AM291,PES1UG24AM262


Project Overview
This project implements a lightweight container runtime in C, inspired by Docker. It includes a long-running supervisor process, CLI-based control, namespace-based isolation, kernel-level memory monitoring, and a bounded-buffer logging system.

The system demonstrates key operating system concepts such as process isolation, inter-process communication (IPC), scheduling, synchronization, and kernel-user space interaction.


Build, Load and Run Instructions

Build:
make

Load Kernel Module:
sudo insmod monitor.ko
sudo dmesg | tail

Verify Device:
ls -l /dev/container_monitor
sudo chmod 666 /dev/container_monitor

Start Supervisor:
sudo ./engine supervisor ./rootfs-base

Start Containers:
sudo ./engine start alpha ./rootfs-alpha "sleep 1000"
sudo ./engine start beta  ./rootfs-beta  "sleep 1000"

List Containers:
sudo ./engine ps

View Logs:
sudo ./engine logs alpha

Memory Monitoring:
gcc -o memory_hog memory_hog.c
cp memory_hog rootfs-alpha/
sudo ./engine start memtest ./rootfs-alpha ./memory_hog
sudo dmesg | tail

Scheduling Experiment:
sudo ./engine start c1 ./rootfs-alpha "yes > /dev/null"
sudo ./engine start c2 ./rootfs-beta  "yes > /dev/null"
top

Clean Teardown:
sudo ./engine stop alpha
sudo ./engine stop beta
sudo rmmod monitor


Architecture Overview

The system consists of three main components:

1. Supervisor (engine.c)
   - Long-running process
   - Manages containers
   - Handles IPC via UNIX socket
   - Maintains container metadata

2. CLI Client
   - Sends commands to supervisor
   - Receives responses
   - Stateless execution

3. Kernel Module (monitor.c)
   - Tracks memory usage of containers
   - Enforces soft and hard limits
   - Communicates using ioctl


Isolation Mechanisms

Isolation is implemented using Linux namespaces:
- PID namespace for process isolation
- UTS namespace for hostname isolation
- Mount namespace for filesystem isolation

Containers are created using:
unshare(CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS);

Filesystem isolation is achieved using:
chroot(abs_rootfs);
chdir("/");

Each container uses its own root filesystem such as rootfs-alpha, rootfs-beta, etc.

All containers share:
- the same kernel
- system memory
- scheduler

This makes containers lightweight compared to virtual machines.


Supervisor and Process Lifecycle

The supervisor manages container lifecycle:
- Creates containers using fork()
- Tracks container metadata (PID, state, limits)
- Handles termination and cleanup

Signal handling:
- SIGCHLD is used to detect container exit
- Prevents zombie processes

Lifecycle flow:
1. CLI sends command
2. Supervisor creates container
3. Metadata stored
4. Process executes
5. SIGCHLD updates state


IPC, Threads and Synchronization

Control Path:
- UNIX domain sockets used for CLI → supervisor communication

Logging Path:
- Pipes used to capture container stdout/stderr

Bounded Buffer:
A producer-consumer model is implemented for logging.

Producer:
- Reads from pipe
- Pushes logs into buffer

Consumer:
- Writes logs to file

Synchronization:
- pthread mutex
- condition variables

Ensures:
- no data loss
- no deadlocks
- safe concurrent logging


Memory Management and Enforcement

Memory usage is measured using RSS (Resident Set Size):
get_mm_rss(mm)

Soft Limit:
- Logs warning when exceeded

Hard Limit:
- Kills process using SIGKILL

Kernel module enforces limits:
- ensures accuracy
- avoids user-space delays
- provides system-level control

Communication:
ioctl(monitor_fd, IOCTL_REGISTER_PID, &entry);


Scheduling Behaviour

Two types of workloads are tested:

CPU-bound:
- yes > /dev/null
- continuously consumes CPU

I/O-bound:
- io_pulse
- periodically yields CPU

Observations:
- Linux uses Completely Fair Scheduler (CFS)
- CPU-bound tasks get longer execution slices
- I/O-bound tasks get higher responsiveness


Design Decisions and Tradeoffs

Namespace Isolation:
- Used clone/unshare
- Simple but not full isolation
- Good for learning OS concepts

Supervisor Architecture:
- Centralized control
- Easier management
- Single point of failure

IPC and Logging:
- UNIX sockets + bounded buffer
- More complex but realistic design

Kernel Monitor:
- Implemented using kernel module
- Requires root privileges
- Enables accurate enforcement

Scheduling Experiments:
- Custom workloads
- Not precise benchmarking
- Demonstrates scheduler behavior clearly


Scheduler Experiment Results

Workload          Behavior
cpu_hog / yes     Continuous CPU usage
io_pulse          Periodic execution

Result:
- Fair CPU distribution
- Better responsiveness for I/O tasks


Key Code Snippets

Container Creation:

pid_t pid = fork();

if (pid == 0) {
    unshare(CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS);
    sethostname(id, strlen(id));

    chroot(abs_rootfs);
    chdir("/");

    mount("proc", "/proc", "proc", 0, NULL);

    char *args[] = { "/bin/sh", "-c", cmd, NULL };
    execvp("/bin/sh", args);
}

Memory Registration:

MonitorEntry me;
me.pid = pid;
me.soft_limit_mb = 50;
me.hard_limit_mb = 100;

ioctl(fd, IOCTL_REGISTER_PID, &me);


Conclusion

This project successfully demonstrates a lightweight container runtime system integrating user-space control with kernel-space enforcement.

It showcases:
- process isolation using namespaces
- supervisor-based container management
- IPC using sockets and pipes
- thread-safe logging
- kernel-level memory monitoring

The implementation provides a practical understanding of how modern container systems like Docker operate internally.
