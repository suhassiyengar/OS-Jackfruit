# OS-Jackfruit: Lightweight Container Runtime

## Team Information
Name: Suhas S Iyengar,Shloka Reddy 
SRN: PES1UG24AM291,PES1UG24AM262

---

## Project Overview
This project implements a lightweight container runtime in C with a supervisor process, CLI interface, namespace-based isolation, and kernel-level memory monitoring.

It demonstrates key operating system concepts such as:
- Process isolation
- Inter-process communication (IPC)
- Scheduling
- Synchronization
- Kernel-user space interaction

---

## Build Instructions
make

---

## Load Kernel Module
sudo insmod monitor.ko
sudo dmesg | tail

---

## Verify Device
ls -l /dev/container_monitor
sudo chmod 666 /dev/container_monitor

---

## Setup Root Filesystem
cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta
cp -a rootfs-base rootfs-gamma

---

## Start Supervisor (Terminal 1)
sudo ./engine supervisor ./rootfs-base

---

## Run Commands (Terminal 2)
cd ~/Desktop/os_jack/boilerplate/OS-Jackfruit/boilerplate

---

## Start Containers
sudo ./engine start alpha ./rootfs-alpha "sleep 1000"
sudo ./engine start beta  ./rootfs-beta  "sleep 1000"
sudo ./engine start gamma ./rootfs-gamma "sleep 1000"

---

## List Containers
sudo ./engine ps

---

## View Logs
sudo ./engine logs alpha

---

## Stop Container
sudo ./engine stop alpha

---

## Memory Monitoring (Soft & Hard Limits)

Create memory_hog.c:
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int main() {
    size_t size = 10 * 1024 * 1024;

    while (1) {
        void *p = malloc(size);
        if (!p) break;
        memset(p, 1, size);
        printf("Allocated 10MB\n");
        sleep(1);
    }
    return 0;
}

Compile:
gcc -o memory_hog memory_hog.c

Copy into container:
cp memory_hog rootfs-alpha/

Run:
sudo ./engine start memtest ./rootfs-alpha ./memory_hog

Check kernel logs:
sudo dmesg | tail -n 30

Expected Output:
[jackfruit] SOFT LIMIT: container 'memtest' ...
[jackfruit] HARD LIMIT: container 'memtest' ... — sending SIGKILL

---

## Scheduling Experiment
sudo ./engine start c1 ./rootfs-alpha "yes > /dev/null"
sudo ./engine start c2 ./rootfs-beta  "yes > /dev/null"
top

---

## Cleanup
sudo ./engine stop alpha
sudo ./engine stop beta
sudo ./engine stop gamma
sudo rmmod monitor

---

## Key Concepts Demonstrated
- Namespace-based isolation
- Supervisor-managed container lifecycle
- IPC using UNIX sockets and pipes
- Bounded-buffer logging (producer-consumer)
- Kernel-level memory enforcement
- Linux scheduling behavior

---

## Notes
- Run supervisor in a separate terminal
- Use sudo for all commands
- Each container must use a separate rootfs
- Use sudo dmesg to view kernel logs

---

## Conclusion
This project demonstrates how container runtimes operate internally by combining user-space control with kernel-space enforcement. It provides practical insight into modern OS concepts and container systems like Docker.
