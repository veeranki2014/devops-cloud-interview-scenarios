# 🐧 Linux / SRE — Scenario-Based Interview Questions

**Q1. [L1] A critical web server is unresponsive, butping still works. SSH connection is refused. What could be the issue and how would you resolve it?**

> *What the interviewer is testing:* Basic system troubleshooting, understanding of SSH vs. ICMP.

**Answer:**
If `ping` works but `ssh` is refused, the server is up (network layer is fine) but the SSH daemon (`sshd`) is either down, rejecting connections (e.g., max connections reached, TCP wrappers), or a local firewall is blocking port 22.

To resolve: If physical or console access (like AWS Systems Manager or a hypervisor console) is available, I would log in and check:
1. Is `sshd` running? `systemctl status sshd`
2. Check logs: `journalctl -u sshd` or `/var/log/auth.log`
3. Check firewall rules: `iptables -L` or `ufw status`
If it's an Out Of Memory (OOM) issue, `sshd` might have been killed by the OOM killer. I'd reboot the instance via the cloud provider console if no other access is possible to restore service, then investigate `/var/log/messages` after recovery.

---

**Q2. [L2] An application deployed on a Linux machine suddenly starts failing with "No space left on device", but `df -h` shows the partition is only 50% full. What is happening?**

> *What the interviewer is testing:* Understanding of Linux filesystems, inodes vs. disk space.

**Answer:**
This is a classic issue of running out of inodes. While the physical disk blocks might be available, the filesystem has exhausted its allocation of inodes, which are required to track every file and directory. This typically happens when an application creates millions of tiny files (like session files, logs, or cache files).

I would verify this by running `df -i`. If `IUse%` is at 100%, I need to find the directory with the massive number of files.
To find the culprit, I can run:
`find / -type d -size +100K` (Directories grow in size when they contain many entries)
Or the slower but precise method:
`for i in /*; do echo $i; find $i |wc -l; done`
Once found (often `/tmp` or `/var/lib/php/sessions`), I would delete or archive the files, potentially using `find <dir> -type f -mtime +7 -delete` to prevent arguments list too long errors.

---

**Q3. [L2] Your team receives PagerDuty alerts that CPU utilization is consistently hitting 100% on a database server. How do you find what's causing it?**

> *What the interviewer is testing:* CPU troubleshooting workflow, differentiating between system, user, and wait CPU.

**Answer:**
First, I'd SSH into the server and run `top` or `htop`. I'd check the load average to see the trend (1, 5, 15 minutes).
In `top`, I'd look at the CPU states:
- High `%us` (user): The application or DB process is working hard. I'd find the PID, then use `strace -p <PID>` or `perf top` or check DB-specific tools (like `SHOW PROCESSLIST` in MySQL) to see what queries are running.
- High `%sy` (system): The kernel is doing heavy work (e.g., millions of context switches or network interrupts).
- High `%wa` (iowait): The CPU is idling waiting for I/O. This means it's a disk bottleneck, not a CPU bottleneck. I'd switch to `iotop` or `iostat -x 1` to find the process thrashing the disk.
- High `%st` (steal): If it's a VM, the noisy neighbor syndrome. The hypervisor is allocating cycles elsewhere. I would migrate the VM or resize the instance.

---

**Q4. [L1] A user complains they cannot execute a script (`./script.sh`), getting a "Permission denied" error. They own the file and have `rwx` permissions. What else could cause this?**

> *What the interviewer is testing:* Filesystem mount options, SELinux/AppArmor, shell interpretation.

**Answer:**
Even if the file has `chmod +x` and the user owns it, execution can be blocked by:
1. **Mount options:** The filesystem where the script resides might be mounted with the `noexec` flag (often used for `/tmp` or `/var/tmp` for security). You can check this by running `mount | grep noexec`.
2. **Interpreter path:** The script's shebang (`#!/bin/bash`) might point to a missing interpreter, or the interpreter itself lacks execute permissions.
3. **SELinux/AppArmor:** Mandatory Access Control policies might block the execution. Checking `dmesg` or `/var/log/audit/audit.log` will reveal SELinux denials.
4. **Improper ACLs:** Extended ACLs (checked via `getfacl script.sh`) might have a deny rule superseding standard POSIX permissions.

---

**Q5. [L2] During an incident, you notice that a specific log file (`/var/log/app.log`) is growing at 5GB per minute, threatening to fill the disk. Deleting the file doesn't free up the disk space space. Why, and what do you do?**

> *What the interviewer is testing:* How Linux handles open file descriptors, disk space recovery without downtime.

**Answer:**
In Linux, when you `rm` a file, you delete the directory entry (the link to the inode). However, the disk blocks are not freed as long as any running process holds an open file descriptor to that file. The application is still writing to the deleted file's inode.

Running `lsof | grep deleted` will show the application holding the file open.
To actually free the space without restarting the application (which might cause downtime), I would truncate the file instead of deleting it. 
If the file is already deleted but held open, I would find it via the proc filesystem and truncate it:
1. Find PID using `lsof`.
2. Find the FD number in `lsof` output (e.g., FD 4).
3. Truncate it: `> /proc/<PID>/fd/4`
In the future, I'd configure `logrotate` to use `copytruncate` or send a `SIGHUP` to the app to force it to reopen logs gracefully.

---

**Q6. [L2] An application tries to bind to port 443 but fails with "Permission denied". The port is not in use. Why is this happening?**

> *What the interviewer is testing:* Linux privileged ports, setcap, port forwarding.

**Answer:**
In Linux, ports below 1024 are considered "privileged ports." Only processes running as `root` can bind to them. For security reasons, web servers and applications are typically run as non-root users.

To fix this without running the application as root, there are three common approaches:
1. **Capabilities (Modern/Best Practice):** Use `setcap` to grant the specific binary permission to bind to privileged ports: `setcap 'cap_net_bind_service=+ep' /path/to/app`.
2. **Reverse Proxy:** Run a reverse proxy like Nginx or HAProxy as root, bound to 443, and forward traffic to the app running on a high port (e.g., 8443) as a normal user.
3. **iptables:** Redirect traffic from port 443 to a high port: `iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port 8443`.

---

**Q7. [L2] A developer accidentally ran `chmod -R 777 /var/www/html` and now the web server is throwing 500 errors. How do you recover?**

> *What the interviewer is testing:* File permissions, understanding of web server security context.

**Answer:**
`777` permissions mean anyone can read, write, and execute the files. Many modern web applications, PHP-FPM, or SSH instances will refuse to read configurations or execute scripts if they are globally writable, citing security risks.

To fix this, we need to revert to standard secure permissions: Directories should typically be `755` and files should be `644`.
I would run:
1. `find /var/www/html -type d -exec chmod 755 {} \;`
2. `find /var/www/html -type f -exec chmod 644 {} \;`
Then, I'd ensure the ownership is correct (e.g., owned by `nginx:nginx` or `www-data:www-data`): `chown -R www-data:www-data /var/www/html`. This restores standard secure execution parameters.

---

**Q8. [L3] The system is experiencing unexplained network latency. `ping` times internally jump from 1ms to 200ms periodically. How do you isolate the problem?**

> *What the interviewer is testing:* Advanced network troubleshooting, packet capture, kernel networking stack.

**Answer:**
Intermittent high latency is tricky. My workflow would be:
1. **Scope:** Is it affecting all instances? One AZ? One specific app? I'd ping the default gateway and another instance in the same subnet to isolate whether it's the host network stack or the physical network/hypervisor.
2. **Host metrics:** Check `dmesg` or `journalctl` for NIC errors, ring buffer drops, or `nf_conntrack` table full messages.
3. **Traffic Analysis:** Use `mtr` to track packet loss at hops. To capture the issue, I'd run `tcpdump -i eth0 -w capture.pcap` during the latency spikes.
4. **Kernel Drops:** Use `netstat -su` to see if UDP/TCP packets are being dropped by the kernel due to full receive buffers. I'd verify `top` to check if a specific CPU core handling NIC interrupts (`ksoftirqd`) is pinned at 100%, causing a processing bottleneck.
Based on the finding, I might tune ring buffers (`ethtool -G`), adjust `sysctl` buffers, or escalate to the cloud provider if the host metrics check out perfectly.

---

**Q9. [L3] A service is being killed randomly every few days by the OOM Killer, but monitoring shows the machine has plenty of free RAM available. Why?**

> *What the interviewer is testing:* Cgroups, NUMA architectures, or memory fragmentation.

**Answer:**
If the host has free memory but a process is still OOM killed, the limitation is artificially imposed or architectural:
1. **Cgroups:** If the process runs in a Docker container or a systemd slice with strict `MemoryLimit=` or `--memory`, the cgroup will trigger the kernel OOM killer on the process specifically when it breaches its localized limit, regardless of host free memory. Check `dmesg -T | grep -i oom`.
2. **NUMA Nodes:** On large multi-socket servers, memory is tied to specific CPU sockets (NUMA nodes). If the application sets strict `numactl` policies to pin memory to a specific node, and *that* node runs out of memory, it may OOM kill rather than fetch memory across the slow QPI link from another socket.
3. **32-bit Architecture constraints:** Though rare nowadays, 32-bit apps max out around 3-4GB of accessible address space.

---

**Q10. [L1] You need to find all `.log` files modified in the last 7 days and copy them to an archive directory. What command do you use?**

> *What the interviewer is testing:* Familiarity with `find` and `xargs` or `-exec`.

**Answer:**
The most efficient command would be:
`find /path/to/search -name "*.log" -mtime -7 -type f -exec cp {} /path/to/archive/ \;`
Alternatively, for a large number of files (to avoid spawning multiple `cp` processes), use `xargs`:
`find /path/to/search -name "*.log" -mtime -7 -type f -print0 | xargs -0 -I {} cp {} /path/to/archive/`

The `-mtime -7` checks for files modified in less than 7 days, `-type f` restricts to files only, and `-print0` with `-0` handles filenames with spaces safely.

---

**Q11. [L3] You are investigating an application crash, but no core dump was generated. How do you ensure core dumps are created for future crashes?**

> *What the interviewer is testing:* Core dump mechanisms, ulimits, systemd.

**Answer:**
By default, core dumps are often disabled in production environments due to disk space and security concerns. To enable them:
1. **ulimit:** Check and set the soft/hard limits for core file size. `ulimit -S -c unlimited` in the script starting the app, or edit `/etc/security/limits.conf` to set `* soft core unlimited`.
2. **Kernel Pattern:** Check where dumps are written via `sysctl kernel.core_pattern`. It should point to a valid directory or a handler like systemd-coredump (e.g., `|/usr/lib/systemd/systemd-coredump %P %u %g %s %t %c %h`).
3. **Systemd:** If run as a systemd service, the unit file must have `LimitCORE=infinity` in the `[Service]` section.
4. **App configuration:** Ensure the application itself doesn't catch the SEGV signal without re-raising it, or hasn't called `prctl` to disable dumpability (e.g., `PR_SET_DUMPABLE`).

---

**Q12. [L2] Our application uses a third-party API. The API vendor claims they are receiving requests, but our app throws timeout errors. How do you prove whether the traffic is leaving our server successfully?**

> *What the interviewer is testing:* Packet capture, `tcpdump` usage.

**Answer:**
To definitively prove if the traffic is leaving the server and if we are receiving a response, I would use `tcpdump`.
I'd run:
`tcpdump -nni eth0 host <vendor-ip> and port 443`
I'd look at the TCP handshake. 
- If I see `SYN` leaving, but no `SYN-ACK` returning, the traffic is being dropped aggressively (firewall, routing issue on the path, or vendor blocking us).
- If the handshake completes, but SSL/TLS negotiation stalls, or we send an HTTP request and get no `PSH` data back, the vendor's application is hanging, proving the fault is on their end.
- If no `SYN` packets appear in tcpdump at all, the issue is internal to the host (e.g., the app is locked up, or local `iptables` rules drop outbound traffic before it hits the wire).

---

**Q13. [L2] A cron job configured to run every midnight is failing, but when you run the script manually as the same user, it works perfectly. Why?**

> *What the interviewer is testing:* Cron environment variations, absolute paths.

**Answer:**
This is almost always an environment variable issue.
When a user logs in interactively, profiles like `~/.bashrc` and `/etc/profile` are sourced, setting up `$PATH`, aliases, and application-specific variables. Cron executes with a heavily restricted, minimal environment (often just `/usr/bin:/bin`).

To fix this:
1. The script should use absolute paths for every command (e.g., `/usr/bin/curl` instead of `curl`).
2. Alternatively, the crontab can explicitly source the environment at execution:
`0 0 * * * . /home/user/.bash_profile; /home/user/script.sh`
3. Any variables needed by the script should be explicitly exported at the top of the script.

---

**Q14. [L3] Your infrastructure team is migrating to a new storage backend. You need to rsync 10TB of data with millions of tiny files between two Linux servers over the network. How do you optimize for speed?**

> *What the interviewer is testing:* Data transfer optimization, parallel processing, bypassing SSH overhead.

**Answer:**
Standard `rsync -avz` over SSH for millions of tiny files is extremely slow. The bottleneck becomes SSH encryption overhead and the serial nature of rsync analyzing one file at a time.
To optimize:
1. **Parallelization:** I would use `find` piped into `xargs` running multiple rsync processes in parallel, or use tools specifically designed for this like `fpsync` or `GNU parallel`.
2. **Disable Compression:** Remove the `-z` flag from rsync. Compressing already compressed files (like images) wastes CPU, and for tiny files, the compression overhead dominates.
3. **Change Cipher:** If I must use SSH, I'd use a faster cipher: `rsync -e "ssh -c aes128-ctr"`.
4. **Use Netcat/Tar (if secure):** In a trusted backend network, pipe a parallel tar stream directly over unencrypted TCP via `nc`. It avoids SSH entirely.
`Sender: tar -cf - folder | nc -l 1234`
`Receiver: nc <sender-ip> 1234 | tar -xf -`

---

**Q15. [L2] You see a process in `D` state in `top`. You use `kill -9` but it doesn't die. Why, and how do you remove it?**

> *What the interviewer is testing:* Uninterruptible sleep, kernel states, hardware/NFS lockups.

**Answer:**
The `D` state stands for "Uninterruptible Sleep". The process is waiting deeply within the kernel for a hardware or I/O operation to complete (often a hard disk or a stalled NFS mount). 
Because it's in the kernel context rather than user space, it ignores all signals, including `SIGKILL` (`kill -9`).

You cannot kill a `D` state process directly. The solutions are:
1. **Fix the I/O block:** If it's a hung NFS mount, forcing an unmount (`umount -f -l`) or restarting the NFS server might free it. If it's a dying hard drive, resolving the hardware fault is necessary.
2. **Reboot:** If the I/O cannot be restored, the only way to clear the process is to reboot the server. Over time, too many `D` state processes will artificially skyrocket the system Load Average until the machine hangs.

---

**Q16. [L1] How do you persistently mount a new EBS/block volume so it survives reboots?**

> *What the interviewer is testing:* `/etc/fstab` configuration, UUIDs vs device names.

**Answer:**
First, I verify the block device name (`lsblk`), format it (`mkfs.ext4 /dev/nvmeXn1`), and mount it temporarily to verify (`mount /dev/nvmeXn1 /data`).
To make it persistent:
I should never use the device name (like `/dev/nvmeXn1`) in `/etc/fstab` because cloud providers do not guarantee device name ordering across reboots. Instead, I use the UUID.
1. Find the UUID: `blkid /dev/nvmeXn1`
2. Add an entry to `/etc/fstab`:
   `UUID=123-abc-456 /data ext4 defaults,nofail 0 2`
3. Test the entry without rebooting: `umount /data` then `mount -a`. If no errors are thrown, the system will safely mount it on the next boot. (The `nofail` option ensures the server still boots if the volume is detached).

---

**Q17. [L3] An incident occurred where a server ran out of memory, but the `oom-killer` didn't trigger, causing a complete kernel hang (hard lockup). How do you configure the kernel to automatically panic and reboot if this happens again?**

> *What the interviewer is testing:* Sysctl tuning, kernel panic parameters, high availability recovery.

**Answer:**
If the system becomes completely unresponsive in a hard lockup or OOM stall, manual intervention is slow. I would configure the kernel to panic and automatically reboot. This allows auto-scaling groups or load balancers to immediately replace or re-route traffic from the failed node.
Using `sysctl`:
1. `kernel.panic = 10` (Reboot 10 seconds after a panic)
2. `vm.panic_on_oom = 1` (Panic if the kernel hits an unresolvable OOM state, rather than trying to kill processes if the killer is disabled or failing)
3. `kernel.hung_task_panic = 1` (Panic if tasks are hung for too long, like the `D` state lockups).
These settings are added to `/etc/sysctl.conf` to persist across reboots, forcing the system to "fail fast" and rely on infrastructure redundancy.

---

**Q18. [L2] A junior admin removed the executable bit from the `chmod` command itself (`chmod -x /bin/chmod`). How do you fix this since you can't use chmod to fix chmod?**

> *What the interviewer is testing:* Deep understanding of Linux dynamic linking and program execution.

**Answer:**
Since `/bin/chmod` is essentially just an ELF binary, you can invoke the dynamic linker directly to execute the file without it needing the execute bit. 
The dynamic linker/loader itself is executable.
` /lib64/ld-linux-x86-64.so.2 /bin/chmod +x /bin/chmod` (path varies by distro).

Alternatively, you could use a language interpreter built into the system that already has execute permissions:
Python: `python -c "import os; os.chmod('/bin/chmod', 0o755)"`
Or simply copy a tool that can set attributes: `cp /bin/ls /tmp/ls; cat /bin/chmod > /tmp/ls` (Wait, this is complex). The dynamic linker or Python approach is the cleanest SRE-level answer.

---

**Q19. [L1] A user wants to safely store passwords and application secrets as environment variables. What is the standard SRE approach to this?**

> *What the interviewer is testing:* Secret management best practices vs. hardcoding.

**Answer:**
Environment variables are an improvement over hardcoded passwords in git, but they are still insecure because they show up in `printenv`, `/proc/<pid>/environ`, and crash dumps.
The SRE best practice is to never store secrets in shell environment variables. Instead, use a centralized Secret Management system (like AWS Secrets Manager, HashiCorp Vault, or Kubernetes Secrets).
The application should use an SDK to fetch the secret into memory dynamically at runtime using an IAM role or temporary token, or secrets should be injected securely by a sidecar/init container directly into a temporary RAM disk (`tmpfs`) file that the application reads once and clears.

---

**Q20. [L2] What is an inode, and why is an inode number identical for two different files?**

> *What the interviewer is testing:* Understanding of hard links and filesystem architecture.

**Answer:**
An inode (index node) is a data structure on a standard Linux filesystem that stores metadata about a file, like its size, permissions, ownership, and pointers to the disk blocks where the data is stored. It does *not* store the file name.

If two different file names (paths) have the exact same inode number (verified via `ls -i`), it means they are **Hard Links**. Hard links act as multiple names pointing to the same underlying inode and data. Because they reference the same inode, changes made to the data under one name are immediately reflected in the other. Hard links cannot span different filesystems.

---

**Q21. [L1] Explain the difference between a Hard Link and a Soft (Symbolic) Link.**

> *What the interviewer is testing:* Filesystem primitives, inodes.

**Answer:**
- **Hard Link:** A hard link is multiple file paths pointing to the exact same underlying inode (data) on the disk. Because they share the same inode, if you delete the original file, the hard link still retains the data perfectly. Hard links cannot span across different disk partitions or filesystems.
- **Soft Link (Symlink):** A soft link is a shortcut. It is a completely separate file with its own unique inode, and its contents simply contain the text path pointing to the original file. If you delete the original file, the soft link becomes broken (a "dangling symlink"). Soft links can point across different filesystems and even to directories.

---

**Q22. [L2] Explain the architectural shift from SysVinit to Systemd in modern Linux distributions. Why is Systemd controversial but widely adopted?**

> *What the interviewer is testing:* OS Initialization, parallel boot, service management.

**Answer:**
**SysVinit** was the legacy approach. It used sequential bash scripts (`/etc/init.d/`) to start services one by one during boot. It was slow, and writing robust bash scripts to track process PIDs for restarts was error-prone.
**Systemd** is a modern, monolithic init system and suite of tools. 
- *Why adopted:* It parallelizes service startup, drastically reducing boot times. It uses declarative `Unit files` instead of complex bash scripts. It natively uses cgroups to cleanly track and kill child processes (fixing the daemon escape problem).
- *Controversy:* The Unix philosophy dictates "do one thing and do it well." Systemd violates this by managing not just the init process, but logging (`journald`), network (`networkd`), cron (`timers`), and DNS (`resolved`), creating a heavy, monolithic dependency chain deep inside the OS layer.

---

**Q23. [L3] A specific application binary is randomly spinning up CPU. You run `strace -p <PID>` to see what system calls it's making, but attaching `strace` slows the application down so severely it almost crashes in production. What modern alternative provides identical visibility without the penalty?**

> *What the interviewer is testing:* The ptrace overhead, introducing eBPF (BCC/bpftrace).

**Answer:**
`strace` relies on the `ptrace` system call. Every single time the application tries to do anything, `ptrace` forces the kernel to pause the application, do a context switch to the `strace` user-space tool, print the log, and context switch back. In heavily trafficked environments, this 10x overhead cripples the app.
The modern SRE alternative is **eBPF** (specifically using tools like `bpftrace` or BCC tools).
eBPF runs a highly optimized, sandboxed program directly inside the kernel. It intercepts the sys-calls securely at the kernel level, aggregates the data in memory, and only passes the summarized findings back to user space asynchronously. The overhead is virtually zero, making it completely safe for profiling high-load production servers.

---

**Q24. [L1] In bash, what does the operator `2>&1` mean, for example in `command > output.log 2>&1`?**

> *What the interviewer is testing:* File descriptors, standard output, standard error.

**Answer:**
This operator manages file descriptors in Linux: `1` is Standard Output (STDOUT) and `2` is Standard Error (STDERR).
When running a command, `> output.log` explicitly redirects STDOUT to the file. However, if the command throws an error, it will still print to the terminal, avoiding the file.
The `2>&1` syntax explicitly tells the shell to redirect file descriptor 2 (STDERR) to the exact same destination as file descriptor 1 (STDOUT). This guarantees that both successful output and failure errors are captured sequentially in `output.log`.

---

**Q25. [L2] What is a "Zombie" process (marked as `Z` in `top`), and why does `kill -9` not remove it?**

> *What the interviewer is testing:* Process lifecycle, parent/child relationships, reaping.

**Answer:**
A **Zombie** process is a process that has already fully terminated and finished its execution, but still occupies an entry in the process table.
It happens when a child process exits, but its Parent Process hasn't read its exit status yet (via the `wait()` system call). The kernel keeps the zombie around just to preserve the exit code.
You cannot `kill -9` a zombie because it is already dead—there is no execution thread left to kill.
To clear it, you must either:
1. Fix the Parent Process so it `waits` properly.
2. If impossible, `kill` the Parent Process. The zombie will then be "orphaned" and automatically adopted by `systemd` (PID 1), which immediately reaps it and clears the process table.

---

**Q26. [L3] A database server has 64GB of RAM and Swap enabled. Swappiness is set to the default of 60. You notice the database occasionally acts sluggish because Linux is using 5GB of Swap, even though 10GB of RAM is perfectly free. You disable Swap entirely via `swapoff -a` to force it entirely into RAM, and sudden OOM Outages begin. Explain this behavior.**

> *What the interviewer is testing:* `vm.swappiness`, Anonymous memory vs Page Cache, memory pressure.

**Answer:**
Linux actively manages memory between Application Memory (Anonymous) and File Caching (Page Cache).
A `swappiness` of 60 aggressively pages out idle, rarely used application memory to the relatively slow Swap partition to free up RAM so Linux can cache heavily requested disk files, speeding up overall I/O. 
When you ran `swapoff`, you ripped away the safety net. Now, when the database makes a sudden, massive memory allocation request, Linux has no Swap to move the idle pages to. It must furiously drop disk caches to find space. If the cache dropping isn't fast enough to satisfy the allocation speed, the kernel enters an Out Of Memory condition and the OOM Killer aggressively terminates the database to save the OS.
*Fix:* Turn Swap back on, but drastically tune `sysctl vm.swappiness=1` (or 10) to instruct the kernel to only swap out memory absolutely as a last resort before an OOM.

---

**Q27. [L1] You type `uptime` and the Load Average says `4.00, 3.50, 3.00`. The server runs a 4-core CPU. Is the server overloaded?**

> *What the interviewer is testing:* Interpreting load average relative to CPU cores.

**Answer:**
No, the server is perfectly saturated, but not overloaded.
Load Average represents the number of processes currently utilizing a CPU core, plus the processes waiting in the queue for a turn.
If you have 4 CPU cores, a load of exactly `4.00` means every single core is 100% busy doing active work, but zero processes are stuck waiting in a queue. It is maximal efficiency.
If the load jumped to `8.00` on a 4-core machine, then 4 processes are running, and 4 are waiting—meaning the CPU is severely bottlenecked and users are experiencing delays.

---

**Q28. [L2] You need to run an untrusted application securely. Explain what a `chroot` jail is and why it shouldn't be your only layer of security.**

> *What the interviewer is testing:* Filesystem isolation vs complete containerization.

**Answer:**
**chroot** (Change Root) modifies the root directory (`/`) for a specific running process and its children. If you `chroot` an app into `/var/jail`, the app believes `/var/jail` is the absolute bottom of the entire filesystem. It physically cannot `cd ../../etc` to steal system passwords.
*Why it's insufficient:* `chroot` *only* isolates the filesystem view. The application can still see all other processes in the host OS, intercept network traffic, and access host hardware. If the app runs as `root` inside the jail, knowledgeable attackers can craft system calls to escape the `chroot` entirely. True isolation requires Linux Namespaces and Cgroups (e.g., Docker containerization), not just `chroot`.

---

**Q29. [L3] During a massive DDoS attack or viral traffic spike, you see thousands of `SYN_RECV` states in `netstat` and the kernel is dropping legitimate connections with a "syn flooding" message. How do you tune the kernel networking stack on the fly to survive this?**

> *What the interviewer is testing:* Tuning `sysctl`, kernel ring buffers, SYN backlog.

**Answer:**
The server is dropping packets because the temporary queue that holds incomplete TCP connections (waiting for the final ACK) is full.
I would dynamically increase the queue limits in the kernel via `sysctl` immediately:
1. `sysctl -w net.ipv4.tcp_max_syn_backlog=4096` (Increases the size of the incomplete connection queue).
2. `sysctl -w net.core.somaxconn=4096` (Increases the absolute limit of socket listen queues).
3. `sysctl -w net.ipv4.tcp_syncookies=1` (Enables SYN Cookies, allowing the kernel to stop saving state for incomplete SYNs entirely, bypassing the backlog limits cryptographically).
These command-line tweaks take effect immediately without a reboot, mitigating the drops live. I would then save them in `/etc/sysctl.conf`.

---

**Q30. [L1] A log file consists of rows like: `ERROR 2023-10-01 User bob failed login`.
Using basic Linux text processing, how would you print only the names (like 'bob') of the users who failed?**

> *What the interviewer is testing:* `awk` usage, stream manipulation.

**Answer:**
I would use `awk`. 
By default, `awk` splits strings by whitespace into numbered variables. 
The command would be: `cat app.log | grep "failed login" | awk '{print $4}'`.
Since "bob" is the 4th word separated by spaces (`ERROR` is $1, date is $2, `User` is $3), it cleanly extracts just the username.

---

**Q31. [L2] A server's disks are incredibly slow. `iostat -x 1` shows %util at 100%. How do you identify whether it's latency on reading data or writing data?**

> *What the interviewer is testing:* Deciphering complex I/O metrics, `iostat` columns.

**Answer:**
While `%util` shows the disk is completely saturated (100% time spent doing I/O), it doesn't explain the load profile.
In the `iostat` output, I would focus on two specific column groups:
- **`r_await` vs `w_await`:** These denote the average wait time (in milliseconds) for Read requests vs Write requests to be served. If `r_await` is 5ms and `w_await` is 500ms, the disk is severely struggling to write.
- **`rkB/s` vs `wkB/s`:** These denote the actual volume of Kilobytes Read vs Written per second.
If the write latency is massive, turning on Write-Back caching on a RAID controller, or migrating to an SSD, is the immediate hardware solution.

---

**Q32. [L3] You need to debug a broken Docker image, but the container instantly crashes upon starting, so you can't `docker exec` into it. You want to inspect the actual files inside the image layer directly on the Linux host disk. Where in the filesystem do you look?**

> *What the interviewer is testing:* The overlay2 filesystem, container storage drivers.

**Answer:**
Docker does not store containers as large VM `.vmdk` files. It uses a Union Filesystem (typically `overlay2`).
All image layers and actively mutated container filesystems exist as raw directories located in `/var/lib/docker/overlay2/`.
To find the exact directory for the broken image:
1. Run `docker inspect <image_tag>`.
2. Look at the `GraphDriver.Data.MergedDir` attribute.
3. CD into that absolute path (`/var/lib/docker/overlay2/abc123.../merged`). 
There, you will see a perfect, unbooted replica of the container's root filesystem ( `/etc`, `/usr`, `/var`). You can use standard Linux tools (`cat`, `vim`) directly on the host to debug the bad config file causing the crash.

---

**Q33. [L1] What is the difference between running `su root` and `su - root`?**

> *What the interviewer is testing:* Linux environments, login shells.

**Answer:**
- `su root` switches your UID privileges to root, but it explicitly **preserves** your existing user's environment. You keep your old `$PATH`, current working directory, and aliased commands.
- `su - root` (with the hyphen) forces a full **login shell**. It completely purges your previous environment, resets environment variables, changes the directory to `/root`, and sources root's `.bash_profile`. This is the vastly safer and preferred method, ensuring you aren't accidentally executing harmful bin paths inherited from the unprivileged user.

---

**Q34. [L2] An ex-employee changed the root password of a critical bare-metal Linux server and left. You have physical access to the server but do not know the password. How do you regain control?**

> *What the interviewer is testing:* GRUB manipulation, Rescue Mode, `rd.break`.

**Answer:**
Since physical access is available, the root password can be reset by interrupting the bootloader (GRUB).
1. Reboot the server. When the GRUB menu appears, press `e` to edit the kernel boot parameters.
2. Find the line starting with `linux` or `linux16`. Append `rd.break` (on RHEL/CentOS) or `init=/bin/bash` (on Debian/Ubuntu) to the end of that line, and press Ctrl+X to boot.
3. This drops you into an emergency root shell *before* the system fully mounts or asks for a password.
4. Mount the actual root filesystem mathematically with read-write access: `mount -o remount,rw /sysroot` and `chroot /sysroot`.
5. Run the `passwd` command to comfortably set a new root password. Exit and reboot.

---

**Q35. [L3] The OS memory is exhausted, but `top` and `ps` show that the sum of all running applications is only using 20% of the RAM. You clear the page cache, but the memory is still gone. Where is the "missing" memory?**

> *What the interviewer is testing:* Kernel Slab allocations, dentry/inode caches, memory leaks in kernel modules.

**Answer:**
If user-space processes and the page cache aren't holding the memory, it is allocated directly within the **Kernel Slab Allocator**. The Slab holds kernel structures like active `dentry` caches (directory paths) or inodes, or it's a literal memory leak caused by a buggy kernel driver.
I would run the `slabtop` command.
If `slabtop` shows the `dentry` cache is gigabytes in size, the server has traversed millions of tiny files recently. While normally the system reclaims this under pressure, sometimes a `sysctl vm.drop_caches=3` is required to force the kernel to let go of directory structures, immediately restoring the "missing" memory.

---

**Q36. [L1] In bash scripting, what is the significance of the exit code `$?`, and what does an exit code of `0` mean?**

> *What the interviewer is testing:* Basic shell scripting, POSIX exit conventions.

**Answer:**
The specialized variable `$?` holds the numeric exit code of the absolute last command executed in the terminal.
In Linux architecture, an exit code of **`0` means success** (the command executed perfectly without errors). 
Any numeric exit code greater than 0 (e.g., `1`, `127`, `255`) means failure. This is why scripts often check `if [ $? -eq 0 ]` to decide whether to proceed to the next step or halt and throw an error.

---

**Q37. [L2] You want to quickly check if a web server process is actively listening on port 443, and see what PID is managing it. Both `netstat` and `ss` commands are available. Which is preferred in modern Linux and why?**

> *What the interviewer is testing:* Modern iproute2 suite vs deprecated net-tools.

**Answer:**
While both achieve the result, **`ss`** (Socket Statistics) is vastly preferred and heavily replaces the deprecated `netstat`.
- **Performance:** `netstat` parses the `/proc/net` files sequentially. On a heavily loaded proxy server with 50,000 connections, `netstat` is notoriously agonizingly slow and resource-heavy. `ss` queries the kernel directly via the faster `netlink` socket API, returning results instantly regardless of scale.
- **Command:** `ss -tulpn | grep 443` (TCP, UDP, Listening only, Process, Numeric IP) will efficiently display the exact listening state and the PID directly.

---

**Q38. [L3] An application developer asks you, "Can I read the hardware CPU temperature and exact memory mappings using simple text files rather than writing a C program?" How does the Linux architecture support this?**

> *What the interviewer is testing:* The `/proc` and `/sys` pseudo-filesystems, virtual files.

**Answer:**
Yes. This is entirely supported by the **`/proc`** and **`/sys`** pseudo-filesystems.
These are not real physical directories stored on the hard drive; they are dynamic illusions generated strictly in RAM by the kernel.
When you run `cat /proc/cpuinfo` or `cat /sys/class/thermal/thermal_zone0/temp`, you are not reading a file; you are actually making a direct system call right into the kernel space. The kernel instantly aggregates the hardware or process data and renders it as plain text output to your terminal, adhering strictly to the Linux philosophy that "everything is a file."

---

**Q39. [L2] A legacy cron job runs every 5 minutes (`*/5 * * * *`). It takes 6 minutes to complete, so instances are overlapping, locking the database, and failing. The developer wants to migrate it to `systemd` timers. How does systemd solve this overlapping issue inherently?**

> *What the interviewer is testing:* Cron vs Systemd Timers, execution safety.

**Answer:**
Traditional cron is notoriously "dumb." It strictly fires by the clock regardless of the state of the previous job, requiring developers to write complex bash `flock` (file locking) wrappers to prevent overlap.
**Systemd Timers** solve this intrinsically. When you migrate it to a `service.timer` unit, the default behavior of systemd is that it **will not trigger** the linked service unit if that specific service is already actively running. It mathematically protects against overlapping executions without needing a single line of locking code written by the developer.

---

**Q40. [L1] A user downloads a custom binary file `kubectl` into their `~/Downloads` folder. They type `kubectl` in the terminal, but get `command not found`. Why, and how do they fix it permanently?**

> *What the interviewer is testing:* The `$PATH` environment variable.

**Answer:**
The shell relies on the **`$PATH`** environment variable, which is an ordered list of directories (like `/usr/bin` or `/usr/local/bin`). When you type a command, the system only searches those exact directories for the executable. It does not actively search `~/Downloads`.
To fix it, they must either:
1. Move the binary into an authorized path: `sudo mv ~/Downloads/kubectl /usr/local/bin/` 
2. Permanently add the Downloads directory to their path by appending `export PATH=$PATH:~/Downloads` to their `~/.bashrc` or `~/.zshrc` profile, so the shell learns to search there automatically.

---

**Q41. [L2] You need to run a long-lived background service that must automatically restart if it crashes. You also want to manage its lifecycle (start/stop/restart) from a single systemd unit file. How do you configure systemd for auto-restart, and what are the critical directives?**

> *What the interviewer is testing:* Systemd service definition, restart policies, type directives.

**Answer:**
Modern Linux systems use **systemd** to manage services instead of SysVinit scripts. A systemd unit file (e.g., `/etc/systemd/system/myapp.service`) must configure restart behavior explicitly.

Key directives:
1. **Type=simple** (default): The process runs in the foreground. Systemd considers the service active when the process starts.
2. **Type=forking**: The process daemonizes itself (old SysVinit behavior). Systemd waits for the parent to exit before considering it active.
3. **Restart=on-failure**: Automatically restart if the process exits with a non-zero exit code.
4. **RestartSec=5**: Wait 5 seconds between restart attempts to avoid restart loops.
5. **StandardOutput=journal** and **StandardError=journal**: Capture all logs directly to journald (no separate log files needed).

Example configuration:
```ini
[Unit]
Description=My Critical Application
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/myapp
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Then enable and start: `systemctl daemon-reload && systemctl enable myapp && systemctl start myapp`.

---

**Q42. [L1] You need to inspect the current state of a running process without attaching external debugging tools. What files in the `/proc` filesystem give you process information like memory usage, open file descriptors, and CPU affinity?**

> *What the interviewer is testing:* Understanding /proc filesystem structure, process introspection.

**Answer:**
The Linux **`/proc` filesystem** exposes the kernel's data structures as text files, allowing deep process inspection without external tools.

For a process with PID `1234`, key files are:
- **`/proc/1234/status`**: Summary of memory usage (VmPeak, VmRSS, VmSwap), process state, UIDs, thread count, etc.
- **`/proc/1234/stat`**: CPU time, process state (`S`=sleeping, `R`=running, `Z`=zombie), PPid, nice value.
- **`/proc/1234/fd/`**: Directory listing all open file descriptors. `ls -la /proc/1234/fd/` shows which files/sockets the process has open.
- **`/proc/1234/maps`**: Virtual memory map showing loaded libraries, heap, stack regions, and their permissions.
- **`/proc/1234/limits`**: Current hard/soft limits (ulimits) for file descriptors, core dump size, CPU time, etc.
- **`/proc/1234/cgroup`**: Which cgroups (namespaces and resource limits) the process belongs to.

Quick example: `cat /proc/1234/status | grep VmRSS` shows exact resident memory used. `cat /proc/1234/limits` shows if file descriptor limits are exhausted.

---

**Q43. [L3] Explain how Linux Namespaces and Cgroups work together to isolate and limit container resources. Why does removing a container process not delete the cgroup, and what happens if you reuse the cgroup name?**

> *What the interviewer is testing:* Container isolation fundamentals, kernel-level resource management.

**Answer:**
**Namespaces** and **Cgroups** are complementary isolation mechanisms:

**Namespaces** isolate *visibility* (PID namespace hides other processes, network namespace creates a separate NIC, filesystem namespace provides a chroot-like view).
**Cgroups** enforce *resource limits* (CPU shares, memory cap, I/O bandwidth, etc.).

When a container starts, the container runtime (Docker/containerd):
1. Creates new namespaces for the process: `unshare(CLONE_NEWPID | CLONE_NEWNET | ...)`
2. Assigns the process to a cgroup: `/sys/fs/cgroup/memory/docker/<container-id>/` with `memory.limit_in_bytes=512M`
3. The container process thinks it's PID 1 (due to PID namespace) but the host sees it as PID 4567.

When the container process terminates, **the cgroup persists** because cgroups are kernel-managed resource accounting structures separate from process lifecycles. The kernel keeps tracking memory usage, I/O metrics, etc., even if the process exits. This allows inspecting historical resource consumption after a crash.

**Reusing the same cgroup name:** If you spin up a new container and reuse the same cgroup path without cleaning up the old one, the new process joins the cgroup with the old memory limit still in place and metrics still aggregating. This can cause unexpected behavior (e.g., the old limit prevents the new container from starting). Container runtimes always clean up cgroups (or create unique names with container IDs) to prevent this collision.

---

**Q44. [L2] An application runs fine, but SELinux denies a critical system call with "Permission Denied" (avc denial). Standard Unix permissions show the file is readable. How do you diagnose and temporarily allow the access?**

> *What the interviewer is testing:* Mandatory Access Control (MAC) vs Discretionary Access Control (DAC), SELinux policy debugging.

**Answer:**
SELinux is a **Mandatory Access Control (MAC)** layer that sits *above* traditional Unix permissions (DAC). Even if a file is `644`, SELinux can block access if the policy says the process's domain is not allowed.

When a denial occurs:
1. **Check the audit log:** `audit2why` or `grep AVC /var/log/audit/audit.log` shows the exact denial. Example:
   ```
   type=AVC msg=... avc: denied { read } for pid=1234 comm="apache" name="config.txt" scontext=system_u:system_r:httpd_t:s0 tcontext=staff_u:object_r:user_home_t:s0
   ```
   This shows `httpd_t` domain trying to read a `user_home_t` file—denied by policy.

2. **Temporary fix (debug mode):** Set SELinux to **Permissive** mode for the domain:
   ```bash
   semanage permissive -a httpd_t
   ```
   This logs denials but allows the process to operate, helping identify which accesses are actually needed.

3. **Permanent fix:** Either:
   - Change the file's SELinux context: `chcon -t httpd_sys_rw_content_t /var/www/config.txt`
   - Write a custom policy module using `audit2allow` to automatically generate rules from audit logs.

---

**Q45. [L1] Explain the difference between SIGTERM and SIGKILL. Why must applications handle SIGTERM gracefully before deployment, and what happens during Kubernetes pod termination?**

> *What the interviewer is testing:* Process signals, graceful shutdown, orchestration lifecycle.

**Answer:**
**SIGTERM (signal 15):** A "polite request" to terminate. The process can catch it, perform cleanup (flush buffers, close connections), and exit cleanly.
**SIGKILL (signal 9):** A "forced kill" that cannot be caught or ignored. The kernel immediately terminates the process, potentially leaving corrupted state.

In production deployments (especially Kubernetes):
1. When scaling down or rolling out updates, the orchestrator sends **SIGTERM** to the application (e.g., 30-second grace period).
2. The application **must** catch SIGTERM and gracefully shutdown:
   ```bash
   trap 'graceful_shutdown' SIGTERM
   ```
3. During shutdown: close client connections, finish in-flight requests, flush logs, release resources.
4. If the process doesn't exit cleanly within the grace period, Kubernetes sends **SIGKILL**, forcefully terminating it (potential data loss or corrupted state).

Applications that don't handle SIGTERM risk:
- Lost in-flight requests
- Corrupted database transactions
- Connection pool exhaustion (clients hang waiting for responses)

Example: A web server catching SIGTERM stops accepting *new* connections but finishes serving existing requests before exiting.

---

**Q46. [L3] A database server shows 100% disk utilization on iostat, but random read latency is acceptable while sequential write latency spikes wildly. How do you distinguish between read bottlenecks and write bottlenecks, and what tools reveal the root cause?**

> *What the interviewer is testing:* Advanced I/O profiling, disk subsystem diagnostics, queue depth.

**Answer:**
Disk latency has two distinct bottlenecks: **read latency** (affecting query performance) and **write latency** (affecting commits/fsync). They require different tools to diagnose.

**Tool 1: iostat -x 1** shows:
- `r/s` and `w/s`: Read and write operations per second (independently measured).
- `rareq-sz` and `wareq-sz`: Average request sizes (small vs large).
- `r_await` and `w_await`: Average latency for reads and writes (in ms).
- `svctm`: Service time (how long the disk takes to serve one request).
- `%util`: Percentage of time the disk is busy (100% = fully saturated).

If `r_await` is 2ms but `w_await` is 500ms, writes are the bottleneck, not reads.

**Tool 2: blktrace + blkparse** captures every disk I/O operation:
```bash
blktrace -d /dev/sda -o - | blkparse -i -
```
Output shows timestamp, operation type (R/W), block address, size, queue depth, and latency for each I/O. You can see if writes are getting stuck behind a large read or being reordered by the scheduler.

**Root causes:**
- High write latency + queue depth > 1: Disk controller or SSD firmware throttling writes (e.g., NAND garbage collection).
- High write latency + queue depth = 1: Single slow write (e.g., journal flush to slow media or encryption overhead).
- Random writes slower than sequential: RAID write-back cache eviction or lack of RAID optimization for random workloads.

---

**Q47. [L2] SSH currently uses password authentication for access to production servers. Describe a certificate-based SSH authentication system for managing access to 1000+ servers without relying on password distribution or key rotation complexity.**

> *What the interviewer is testing:* SSH security architecture, certificate-based auth, key management at scale.

**Answer:**
**Certificate-based SSH** (using OpenSSH certificates) eliminates password management and simplifies key rotation across large deployments.

**Architecture:**
1. **CA Setup:** Designate a secure certificate authority machine that signs user and host SSH keys. Generate a CA key pair:
   ```bash
   ssh-keygen -t rsa -f /path/to/ca/ssh_ca
   ```

2. **User Authentication:** Instead of distributing individual keys, users request a signed certificate:
   - User generates their personal key: `ssh-keygen -t rsa -f ~/.ssh/id_rsa`
   - User submits their public key to the CA (via secure API or admin approval workflow)
   - CA signs it with a time-limited validity: `ssh-keygen -s /path/to/ca/ssh_ca -I user@example.com -n username -V +52w ~/.ssh/id_rsa.pub`
   - User receives a `~/.ssh/id_rsa-cert.pub` valid for 52 weeks
   - User connects: `ssh -i ~/.ssh/id_rsa user@production-server`

3. **Server Configuration:** Add to `/etc/ssh/sshd_config` on all 1000 servers:
   ```
   TrustedUserCAKeys /etc/ssh/user_ca.pub
   HostKey /etc/ssh/ssh_host_rsa_key
   HostCertificate /etc/ssh/ssh_host_rsa_key-cert.pub
   ```

4. **Benefits:**
   - No password distribution or memorization
   - Automatic expiration (no need to revoke individual keys; certificates naturally expire)
   - Centralized revocation list (CRL) if needed
   - Audit trail: Each certificate embeds username, timestamp, and approval details
   - Zero-trust: Revoke CA instantly, all certificates become invalid without manual key cleanup

**Operational Scale:** With 1000 servers, update `/etc/ssh/user_ca.pub` once on all servers (via configuration management like Ansible), and cert-based auth works network-wide without touching private keys.

---

**Q48. [L2] A production server is running out of disk space on the root partition, but the disk has unallocated space. The root partition uses LVM. How do you extend the root filesystem online without downtime?**

> *What the interviewer is testing:* LVM management, online filesystem resize.

**Answer:**
LVM (Logical Volume Manager) allows dynamic volume resizing without unmounting or rebooting.

Steps to extend the root filesystem:
1. **Check available space in the volume group:** `vgdisplay` shows free Physical Extents (PE) in the VG.
2. **Extend the logical volume:** `lvextend -l +100%FREE /dev/mapper/rootvg-rootlv` (uses all free space).
3. **Resize the filesystem online:**
   - For ext4: `resize2fs /dev/mapper/rootvg-rootlv`
   - For XFS: `xfs_growfs /`
4. **Verify:** `df -h /` should now show the extended size.

If the Volume Group itself has no free space but the physical disk has unpartitioned space, first create a new partition (`fdisk` or `parted`), create a Physical Volume (`pvcreate /dev/sdaX`), extend the VG (`vgextend rootvg /dev/sdaX`), then proceed with `lvextend`.

---

**Q49. [L3] Your application maintains persistent TCP connections to a backend service. The backend silently crashes and restarts behind a load balancer, but the application continues to hold stale connections and times out. How do you configure TCP keepalive at the kernel level to detect dead connections faster?**

> *What the interviewer is testing:* TCP keepalive tuning, sysctl networking, connection health.

**Answer:**
TCP keepalive is a kernel-level mechanism that sends periodic probe packets on idle connections to verify the remote end is still alive.

The default Linux settings are extremely conservative:
- `net.ipv4.tcp_keepalive_time = 7200` (Wait 2 hours before sending the first probe)
- `net.ipv4.tcp_keepalive_intvl = 75` (75 seconds between probes)
- `net.ipv4.tcp_keepalive_probes = 9` (Send 9 probes before declaring dead)

With defaults, it takes **2 hours + 9×75s = ~2h11m** to detect a dead connection. For production, I would aggressively tune:
```bash
sysctl -w net.ipv4.tcp_keepalive_time=60
sysctl -w net.ipv4.tcp_keepalive_intvl=10
sysctl -w net.ipv4.tcp_keepalive_probes=6
```
Now dead connections are detected in **60 + 6×10 = 120 seconds**. These settings are persisted in `/etc/sysctl.conf`. Note: the application must also enable `SO_KEEPALIVE` on its sockets for these kernel settings to take effect.

---

**Q50. [L1] You start a long-running process in an SSH session, but you need to disconnect without killing the process. What are your options?**

> *What the interviewer is testing:* Process management, SIGHUP, nohup, tmux/screen.

**Answer:**
When an SSH session disconnects, the shell sends `SIGHUP` (Hangup signal) to all child processes, which terminates them by default.

To prevent this:
1. **`nohup`:** Run the command with `nohup ./long_process.sh &`. This explicitly ignores SIGHUP, and output is redirected to `nohup.out`.
2. **`disown`:** If the process is already running, background it with `Ctrl+Z` then `bg`, then run `disown -h %1` to remove it from the shell's job table so it won't receive SIGHUP.
3. **`tmux` or `screen` (Best Practice):** Start a terminal multiplexer session (`tmux new -s mysession`), run the command inside it, then detach (`Ctrl+B, D`). Reconnect later with `tmux attach -t mysession`. The session persists independently of SSH.

For production workflows, `tmux`/`screen` is vastly preferred because you can reattach and see the output interactively.

---

**Q51. [L2] After a security audit, you need to enforce password complexity and account lockout policies across all Linux servers. What mechanism controls this, and how do you configure it?**

> *What the interviewer is testing:* PAM (Pluggable Authentication Modules), security hardening.

**Answer:**
Linux uses **PAM (Pluggable Authentication Modules)** for authentication policy enforcement. PAM configuration files are located in `/etc/pam.d/` and control how authentication, password changes, and session management behave.

Key configurations:
1. **Password Complexity:** Edit `/etc/pam.d/common-password` (Debian) or `/etc/pam.d/system-auth` (RHEL) and add:
   ```
   password requisite pam_pwquality.so retry=3 minlen=12 dcredit=-1 ucredit=-1 ocredit=-1 lcredit=-1
   ```
   This enforces minimum 12 characters with at least 1 digit, 1 uppercase, 1 lowercase, and 1 special character.

2. **Account Lockout:** Add to `/etc/pam.d/common-auth`:
   ```
   auth required pam_faillock.so preauth deny=5 unlock_time=900
   auth required pam_faillock.so authfail deny=5 unlock_time=900
   ```
   This locks the account for 15 minutes after 5 failed login attempts.

3. **Password History:** Prevent password reuse:
   ```
   password required pam_pwhistory.so remember=12
   ```

---

**Q52. [L1] You need to open port 8080 on a CentOS/RHEL server running `firewalld` instead of `iptables`. How do you add the rule permanently?**

> *What the interviewer is testing:* Modern firewall management with firewalld, zones.

**Answer:**
`firewalld` is the default firewall manager on modern RHEL/CentOS/Fedora systems, replacing direct `iptables` manipulation. It uses the concept of **zones** (groups of rules applied to network interfaces).

To open port 8080 permanently:
1. **Add the port to the default zone:**
   ```bash
   firewall-cmd --permanent --add-port=8080/tcp
   ```
2. **Reload the firewall to apply:**
   ```bash
   firewall-cmd --reload
   ```
3. **Verify:**
   ```bash
   firewall-cmd --list-ports
   ```

If you need to restrict the port to a specific source IP range:
```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.0/8" port protocol="tcp" port="8080" accept'
```
The `--permanent` flag is critical—without it, the rule is temporary and lost on reboot or reload.

---

**Q53. [L2] The `journald` logs on a server are consuming excessive disk space. How do you configure log rotation and size limits for `systemd-journald`?**

> *What the interviewer is testing:* Systemd journal management, log retention policies.

**Answer:**
`systemd-journald` stores logs in binary format under `/var/log/journal/` (persistent) or `/run/log/journal/` (volatile, lost on reboot).

To control disk usage, edit `/etc/systemd/journald.conf`:
```ini
[Journal]
SystemMaxUse=500M
SystemKeepFree=1G
SystemMaxFileSize=50M
MaxRetentionSec=30day
```

Key directives:
- **`SystemMaxUse=500M`**: Maximum total disk space the journal can use.
- **`SystemKeepFree=1G`**: Always keep at least 1GB free on the partition.
- **`SystemMaxFileSize=50M`**: Maximum size of individual journal files before rotation.
- **`MaxRetentionSec=30day`**: Delete entries older than 30 days.

Apply immediately: `systemctl restart systemd-journald`

To manually vacuum old logs right now:
```bash
journalctl --vacuum-size=500M
journalctl --vacuum-time=30d
```

---

**Q54. [L3] A production database server uses a hardware RAID-10 array. One disk fails, and the RAID begins rebuilding onto a hot spare. During the rebuild, I/O performance degrades severely. Why does this happen, and how do you mitigate it?**

> *What the interviewer is testing:* RAID rebuild mechanics, I/O prioritization, write-hole problem.

**Answer:**
During a RAID rebuild, the controller must read data from the surviving disks to reconstruct the missing disk's data onto the hot spare. This rebuild I/O competes directly with production I/O for the same physical disk heads, bus bandwidth, and controller queue slots.

Why performance degrades:
1. **Read amplification:** Every block on the surviving disks must be read to reconstruct parity/mirror data—even blocks the application doesn't need.
2. **Contention:** The rebuild process and production queries fight for disk seek time, especially on spinning HDDs.
3. **Write penalty:** RAID-10 mirrors writes, so the rebuild adds extra write operations.

Mitigation strategies:
1. **Throttle rebuild speed:** Most RAID controllers allow setting rebuild priority (e.g., `megacli -AdpSetProp RebuildRate -val 30 -a0` sets it to 30%). Lower values reduce rebuild impact but increase the vulnerable window.
2. **Schedule rebuilds during low traffic:** If possible, initiate manual rebuilds during off-peak hours.
3. **Use SSDs:** SSDs eliminate seek-time contention, making rebuild impact nearly negligible.
4. **RAID controller cache:** Ensure the battery-backed write cache is functioning—it absorbs write bursts during rebuild.

---

**Q55. [L2] A newly compiled kernel module is not loading, with `modprobe` returning "module not found." The `.ko` file exists. What is wrong?**

> *What the interviewer is testing:* Kernel module system, depmod, module paths.

**Answer:**
Linux kernel modules must reside in the correct directory and be indexed by `depmod` before `modprobe` can find them.

Common causes:
1. **Wrong directory:** Modules must be placed under `/lib/modules/$(uname -r)/`. If the `.ko` file was compiled for a different kernel version, it will be in the wrong directory tree.
2. **Missing dependency index:** After placing the module in the correct path, you must run `depmod -a` to rebuild the module dependency database (`modules.dep`). Without this, `modprobe` cannot locate the module.
3. **Version mismatch:** The module was compiled against a different kernel version. The kernel enforces version magic matching. Check with `modinfo mymodule.ko` and compare `vermagic` against `uname -r`.
4. **Force loading (dangerous):** `insmod /path/to/mymodule.ko` bypasses dependency resolution and loads the module directly, but risks kernel panics if dependencies are unmet.

Correct workflow: Place module → `depmod -a` → `modprobe mymodule`.

---

**Q56. [L1] A developer adds an entry to `/etc/hosts` to override DNS for testing, but it doesn't seem to work. DNS still resolves the original IP. What could be wrong?**

> *What the interviewer is testing:* Name resolution order, nsswitch.conf.

**Answer:**
The `/etc/hosts` file provides static hostname-to-IP mappings. Whether it takes precedence over DNS is controlled by `/etc/nsswitch.conf`.

Check the `hosts:` line in `/etc/nsswitch.conf`:
```
hosts: files dns
```
This means the system checks `/etc/hosts` first (`files`), then DNS. If the order is reversed (`hosts: dns files`), DNS takes priority and `/etc/hosts` is only consulted if DNS fails.

Other common issues:
1. **Typo or formatting error:** Each line must be `IP_ADDRESS hostname [alias]`, separated by spaces or tabs. Extra characters or wrong formatting silently breaks the entry.
2. **Application-level DNS caching:** Some applications (Java, browsers) cache DNS lookups internally and ignore `/etc/hosts` changes until restarted.
3. **systemd-resolved:** Modern distributions may route DNS through `systemd-resolved`, which may have its own caching. Flush with `resolvectl flush-caches`.

---

**Q57. [L2] A batch processing job is consuming all available CPU, starving other services on the same server. How do you lower its CPU priority without killing it?**

> *What the interviewer is testing:* Process scheduling, nice/renice, cgroups.

**Answer:**
Linux uses a **nice value** (ranging from -20 to +19) to influence CPU scheduling priority. Higher nice values mean lower priority (the process is "nicer" to others).

To reduce priority of a running process:
1. **`renice`:** `renice +15 -p <PID>` sets the process to a low priority, allowing other processes to get CPU time first.
2. **Starting with low priority:** `nice -n 15 ./batch_job.sh` launches the process with reduced priority from the start.

For stricter enforcement, use **cgroups**:
```bash
systemd-run --scope --property=CPUQuota=25% ./batch_job.sh
```
This limits the process to 25% of a single CPU core regardless of system load—a hard cap rather than a scheduling hint.

The difference: `nice` is a *soft* priority hint that only matters under contention. Cgroups provide a *hard* resource limit that is enforced regardless of whether other processes need CPU.

---

**Q58. [L3] After a security breach, you need to audit all file access and command execution on a Linux server. What kernel-level auditing framework is available, and how do you set it up?**

> *What the interviewer is testing:* Linux Audit Framework (auditd), security forensics.

**Answer:**
The **Linux Audit Framework** (`auditd`) is a kernel-level subsystem that records system calls, file access, and user actions for security compliance and forensics.

Setup:
1. **Install and enable:**
   ```bash
   yum install audit
   systemctl enable --now auditd
   ```

2. **Add audit rules** in `/etc/audit/rules.d/audit.rules`:
   ```bash
   # Monitor changes to /etc/passwd
   -w /etc/passwd -p wa -k identity_change

   # Monitor all execve system calls (command execution)
   -a always,exit -F arch=b64 -S execve -k command_exec

   # Monitor file deletions
   -a always,exit -F arch=b64 -S unlinkat -S unlink -k file_delete

   # Monitor privilege escalation
   -w /usr/bin/sudo -p x -k privilege_escalation
   ```

3. **Load rules:** `augenrules --load`

4. **Search audit logs:**
   ```bash
   ausearch -k identity_change --start today
   aureport --summary
   ```

The audit log is stored in `/var/log/audit/audit.log` and can be forwarded to a SIEM (like Splunk or ELK) for centralized analysis. For compliance standards like PCI-DSS or SOC2, `auditd` is often a mandatory requirement.

---

**Q59. [L1] What is a `tmpfs` filesystem, and why would you use it in production?**

> *What the interviewer is testing:* RAM-backed filesystems, volatile storage use cases.

**Answer:**
**`tmpfs`** is a filesystem that lives entirely in RAM (and optionally Swap). Files stored in `tmpfs` are never written to physical disk, making read/write operations extremely fast but **volatile**—all data is lost on reboot.

Common production use cases:
1. **`/tmp` and `/run`:** Many distributions mount these as `tmpfs` to speed up temporary file operations and automatically clean them on reboot.
2. **Sensitive secrets:** Store decrypted secrets or API keys in `tmpfs` so they are never persisted to disk, reducing attack surface.
3. **Build caches:** Compilation temp files placed in `tmpfs` dramatically speed up build times.
4. **Session data:** Web server session files written to `tmpfs` avoid disk I/O bottlenecks.

To create a `tmpfs` mount:
```bash
mount -t tmpfs -o size=512M tmpfs /mnt/ramdisk
```

To persist across reboots, add to `/etc/fstab`:
```
tmpfs /mnt/ramdisk tmpfs size=512M,noexec,nosuid 0 0
```

---

**Q60. [L2] A server has two network interfaces but traffic to a specific subnet is going out the wrong interface. How do you diagnose and fix the routing?**

> *What the interviewer is testing:* Linux routing table, ip route, policy-based routing.

**Answer:**
Linux maintains a routing table that determines which interface handles traffic to each destination.

Diagnosis:
1. **View the routing table:** `ip route show` displays all routes, including default gateway and subnet-specific routes.
2. **Check which route applies:** `ip route get 10.20.30.40` shows exactly which interface and gateway will be used for a specific destination.

Fix:
1. **Add a specific route:**
   ```bash
   ip route add 10.20.0.0/16 via 192.168.1.1 dev eth1
   ```
   This forces all traffic to `10.20.0.0/16` through `eth1` via gateway `192.168.1.1`.

2. **Delete the incorrect route:**
   ```bash
   ip route del 10.20.0.0/16 via 192.168.2.1 dev eth0
   ```

3. **Persist the route:** Add to `/etc/sysconfig/network-scripts/route-eth1` (RHEL) or `/etc/netplan/*.yaml` (Ubuntu):
   ```
   10.20.0.0/16 via 192.168.1.1 dev eth1
   ```

For complex multi-homed setups, Linux **policy-based routing** with `ip rule` allows routing decisions based on source IP, not just destination.

---

**Q61. [L1] How does `xargs` work, and why would you use it instead of a `for` loop?**

> *What the interviewer is testing:* Efficient command-line pipelining, argument batching.

**Answer:**
**`xargs`** reads items from standard input and executes a command with those items as arguments. It is far more efficient than a `for` loop because it batches multiple arguments into a single command execution rather than spawning a new process for each item.

Example — deleting thousands of files:
- **Slow (for loop):** `for f in $(find . -name "*.tmp"); do rm "$f"; done` — spawns a separate `rm` process per file.
- **Fast (xargs):** `find . -name "*.tmp" -print0 | xargs -0 rm` — feeds hundreds of filenames to a single `rm` invocation.

Key flags:
- **`-0`**: Use null delimiter (handles filenames with spaces safely, paired with `find -print0`).
- **`-P 4`**: Run 4 parallel processes for CPU-intensive operations.
- **`-I {}`**: Replace `{}` with each argument: `cat urls.txt | xargs -I {} curl {}`.

The performance difference is dramatic when processing thousands of items—`xargs` can be 10–50x faster than a naive loop.

---

**Q62. [L2] A junior admin needs root access for specific commands only (restarting services, reading logs), not full unrestricted root access. How do you configure granular sudo access?**

> *What the interviewer is testing:* Sudoers configuration, principle of least privilege.

**Answer:**
The `/etc/sudoers` file (edited exclusively via `visudo` to prevent syntax errors) controls which users can run which commands as root.

To grant granular access:
```bash
# Allow user 'deploy' to restart nginx and read syslog only
deploy ALL=(root) NOPASSWD: /bin/systemctl restart nginx, /bin/systemctl status nginx, /usr/bin/journalctl -u nginx
```

Key directives:
- **`ALL=(root)`**: Can execute on any host as the root user.
- **`NOPASSWD:`**: Don't prompt for password (useful for automation).
- **Specific commands:** Only the exact commands listed are allowed. Running `sudo bash` or `sudo su` would be denied.

Best practices:
1. **Use command aliases** for grouping:
   ```
   Cmnd_Alias WEB_MGMT = /bin/systemctl restart nginx, /bin/systemctl reload nginx
   deploy ALL=(root) NOPASSWD: WEB_MGMT
   ```
2. **Use group-based rules:** `%devops ALL=(root) NOPASSWD: WEB_MGMT` applies to all users in the `devops` group.
3. **Never use `ALL=(ALL) ALL`** for non-admin users—it grants full unrestricted root access.

---

**Q63. [L1] What does the `/etc/shadow` file contain, and why is it separated from `/etc/passwd`?**

> *What the interviewer is testing:* Linux authentication files, password security.

**Answer:**
**`/etc/passwd`** contains user account information (username, UID, GID, home directory, shell) and historically stored password hashes. However, `/etc/passwd` must be world-readable because many programs need to map UIDs to usernames.

**`/etc/shadow`** was introduced to separate the sensitive password hashes into a file that is readable **only by root** (permissions `640` or `600`). This prevents unprivileged users from reading hashed passwords and attempting offline brute-force attacks.

Each line in `/etc/shadow` contains:
- Username
- Hashed password (using algorithms like SHA-512, prefixed with `$6$`)
- Date of last password change (in days since epoch)
- Minimum days between password changes
- Maximum password age
- Warning period before expiry
- Account expiration date

To check password aging for a user: `chage -l username`. To force a password change on next login: `chage -d 0 username`.

---

**Q64. [L2] You need to schedule a one-time task to run at 3:00 AM tonight (e.g., a database maintenance script). How does the `at` command differ from `cron` for this use case?**

> *What the interviewer is testing:* at vs cron, one-time vs recurring scheduling.

**Answer:**
**`cron`** is designed for recurring scheduled tasks (e.g., every day at midnight). **`at`** is designed for one-time scheduled execution—it runs the command once and the job is automatically removed.

Using `at`:
```bash
echo "/usr/local/bin/db-maintenance.sh" | at 03:00
```
Or interactively:
```bash
at 03:00
> /usr/local/bin/db-maintenance.sh
> Ctrl+D
```

Key differences:
| Feature | `cron` | `at` |
|---------|--------|------|
| Frequency | Recurring | One-time only |
| Configuration | Crontab file | Queue-based |
| Environment | Minimal | Captures current env |
| Management | `crontab -l` | `atq` (list), `atrm` (remove) |

Advantage of `at`: It captures the user's current environment at scheduling time, avoiding the environment variable pitfalls common with `cron`. The `atd` daemon must be running for `at` jobs to execute.

---

**Q65. [L3] A production server requires high network availability. You have two physical NICs and need to configure network bonding (NIC teaming) for redundancy. Explain the bonding modes and which one you'd choose for a production database server.**

> *What the interviewer is testing:* Network bonding/teaming, high availability, NIC redundancy.

**Answer:**
Linux **network bonding** combines multiple physical NICs into a single logical interface for redundancy and/or throughput.

Common bonding modes:
- **Mode 0 (balance-rr):** Round-robin — packets are distributed across NICs sequentially. Increases throughput but requires switch configuration (EtherChannel/LACP).
- **Mode 1 (active-backup):** Only one NIC is active; the standby takes over if the active fails. No switch configuration needed. **Best for database servers** where reliability matters more than throughput.
- **Mode 2 (balance-xor):** XOR-based distribution. Requires switch support.
- **Mode 4 (802.3ad / LACP):** Dynamic link aggregation using the IEEE standard. Provides both redundancy and increased throughput. Requires switch LACP support.
- **Mode 6 (balance-alb):** Adaptive load balancing. No switch configuration needed, balances outgoing traffic.

For a production database server, **Mode 1 (active-backup)** is the safest choice:
```bash
nmcli con add type bond con-name bond0 ifname bond0 bond.options "mode=active-backup,miimon=100"
nmcli con add type ethernet con-name bond0-slave1 ifname eth0 master bond0
nmcli con add type ethernet con-name bond0-slave2 ifname eth1 master bond0
```
The `miimon=100` parameter checks link health every 100ms for fast failover.

---

**Q66. [L1] After a kernel upgrade, the server boots into the old kernel instead of the new one. How do you fix this?**

> *What the interviewer is testing:* GRUB bootloader configuration, default kernel selection.

**Answer:**
The GRUB bootloader controls which kernel is loaded at boot. After a kernel upgrade, the default boot entry may still point to the old kernel.

To fix:
1. **List installed kernels:**
   ```bash
   grubby --default-kernel    # Shows current default
   awk -F\' '/menuentry/ {print $2}' /boot/grub2/grub.cfg    # Lists all entries
   ```

2. **Set the new kernel as default:**
   ```bash
   grubby --set-default /boot/vmlinuz-5.14.0-362.el9.x86_64
   ```
   Or by menu index:
   ```bash
   grub2-set-default 0    # First entry (usually the newest kernel)
   ```

3. **Regenerate GRUB config (if needed):**
   ```bash
   grub2-mkconfig -o /boot/grub2/grub.cfg
   ```

4. **Verify and reboot:**
   ```bash
   grubby --default-kernel
   reboot
   ```

After reboot, confirm with `uname -r` that the new kernel is running.

---

**Q67. [L2] Application logs are written to syslog, but the logs for your specific application are mixed with system messages in `/var/log/messages`. How do you configure rsyslog to send your application's logs to a separate file?**

> *What the interviewer is testing:* Syslog configuration, log routing, rsyslog filters.

**Answer:**
**rsyslog** is the standard syslog implementation on most Linux distributions. It uses rules in `/etc/rsyslog.conf` or `/etc/rsyslog.d/*.conf` to route log messages based on facility, severity, or program name.

To separate your application's logs:
1. **Create a filter rule** in `/etc/rsyslog.d/myapp.conf`:
   ```
   # Route all messages from 'myapp' to its own file
   if $programname == 'myapp' then /var/log/myapp.log
   & stop
   ```
   The `& stop` directive prevents the message from also being written to `/var/log/messages`.

2. **Alternative using facility:**
   If the app uses `local0` facility:
   ```
   local0.*    /var/log/myapp.log
   ```

3. **Restart rsyslog:** `systemctl restart rsyslog`

4. **Set up logrotate** for the new file in `/etc/logrotate.d/myapp`:
   ```
   /var/log/myapp.log {
       daily
       rotate 14
       compress
       missingok
       notifempty
       postrotate
           /usr/bin/systemctl reload rsyslog > /dev/null 2>&1 || true
       endscript
   }
   ```

---

**Q68. [L2] The `du -sh /` and `df -h` commands show significantly different disk usage values. `df` shows 90% used, but `du` reports only 60% of the disk in use. What explains this discrepancy?**

> *What the interviewer is testing:* Understanding of deleted-but-open files, reserved blocks, filesystem internals.

**Answer:**
The `du` and `df` commands measure disk usage differently:
- **`du`** walks the directory tree and sums up the sizes of files it can see (existing directory entries).
- **`df`** queries the filesystem's superblock, which tracks total allocated vs. free blocks.

Common causes of discrepancy:
1. **Deleted files still held open by processes:** When a file is deleted (`rm`) but a process still has it open, `du` can't see it (directory entry is gone), but `df` still counts the blocks as used. Fix: `lsof +L1` finds files with zero links (deleted but open). Restart the process or truncate via `/proc/<PID>/fd/<FD>`.
2. **Reserved blocks:** ext4 reserves 5% of the partition for root by default. `df` accounts for these reserved blocks; `du` doesn't know about them. Check with `tune2fs -l /dev/sda1 | grep "Reserved block count"`. Reduce if needed: `tune2fs -m 1 /dev/sda1` (sets reserved to 1%).
3. **Filesystem journal:** The ext4 journal consumes space visible to `df` but not `du`.

---

**Q69. [L1] What is `/dev/null`, and how is it commonly used in shell scripting?**

> *What the interviewer is testing:* Special device files, output suppression.

**Answer:**
**`/dev/null`** is a special virtual device file known as the "bit bucket" or "black hole." Any data written to it is silently discarded, and reading from it immediately returns EOF (End of File).

Common uses:
1. **Suppress output:** `command > /dev/null 2>&1` — discards both STDOUT and STDERR, running the command silently.
2. **Suppress only errors:** `command 2>/dev/null` — hides error messages while showing normal output.
3. **Empty a file:** `cat /dev/null > /var/log/bigfile.log` — truncates the file to zero bytes without deleting the inode (keeping processes that have it open happy).
4. **Test write speed:** `dd if=/dev/zero of=/dev/null bs=1M count=1000` — benchmarks pure throughput without disk involvement.
5. **Conditional checks:** `if grep -q "pattern" file > /dev/null 2>&1; then ...` — tests for a pattern without displaying matches.

`/dev/null` is one of the "everything is a file" Unix primitives that makes shell scripting composable and clean.

---

**Q70. [L2] Standard Unix permissions (owner/group/other) are insufficient for your use case. You need one specific user to have read access to a file without changing the file's owner or group. How do you accomplish this?**

> *What the interviewer is testing:* POSIX Access Control Lists (ACLs), fine-grained permissions.

**Answer:**
**POSIX ACLs (Access Control Lists)** extend the traditional owner/group/other permission model by allowing per-user and per-group rules on files and directories.

To grant a specific user read access:
```bash
setfacl -m u:alice:r /var/data/report.csv
```

To verify:
```bash
getfacl /var/data/report.csv
```
Output:
```
# file: var/data/report.csv
# owner: bob
# group: engineering
user::rw-
user:alice:r--
group::r--
mask::r--
other::---
```

Key commands:
- **Set ACL:** `setfacl -m u:username:rwx file`
- **Set default ACL (inherited by new files):** `setfacl -d -m u:alice:rx /var/data/`
- **Remove ACL:** `setfacl -x u:alice file`
- **Remove all ACLs:** `setfacl -b file`

Note: The filesystem must be mounted with ACL support (most modern ext4/XFS filesystems enable this by default). A `+` symbol appears in `ls -l` output when ACLs are set (e.g., `-rw-r--r--+`).

---

**Q71. [L1] What is the difference between `screen` and `tmux`? When would you choose one over the other?**

> *What the interviewer is testing:* Terminal multiplexers, remote session persistence.

**Answer:**
Both `screen` and `tmux` are terminal multiplexers that allow you to create persistent terminal sessions that survive SSH disconnections.

| Feature | `screen` | `tmux` |
|---------|----------|--------|
| Age | Older (1987) | Newer (2007) |
| Split panes | Basic | Excellent (horizontal + vertical) |
| Scripting | Limited | Highly scriptable |
| Configuration | `.screenrc` | `.tmux.conf` |
| Status bar | Basic | Fully customizable |
| Session sharing | Supported | Easier multi-user support |
| Copy mode | Clunky | Vi/Emacs-style key bindings |

**Choose `screen`** when:
- It's the only multiplexer installed (common on minimal/legacy servers).
- You need a quick single-session persistence (`screen -S mysession`).

**Choose `tmux`** when:
- You need split panes for monitoring multiple log streams.
- You want scriptable session layouts for complex environments.
- You need better copy-paste and scrollback capabilities.

In modern SRE practice, `tmux` is the standard recommendation. Both solve the fundamental problem of SSH session persistence for long-running operations.

---

**Q72. [L3] You need to perform a kernel upgrade on a production server with zero unplanned downtime. The server runs on RHEL/CentOS. Describe the safest approach.**

> *What the interviewer is testing:* Kernel lifecycle management, risk mitigation, rollback strategy.

**Answer:**
Kernel upgrades inherently require a reboot on standard Linux (unlike kpatch/livepatch for security fixes). The goal is to minimize risk and ensure rollback capability.

Safe upgrade workflow:
1. **Pre-flight checks:**
   - Verify current kernel: `uname -r`
   - Check available updates: `yum list kernel`
   - Review changelog for breaking changes: `yum updateinfo info kernel`
   - Ensure GRUB has the old kernel as fallback

2. **Install (don't update) the new kernel:**
   ```bash
   yum install kernel-5.14.0-362.el9
   ```
   This installs alongside the old kernel rather than replacing it.

3. **Verify boot configuration:**
   ```bash
   grubby --default-kernel    # Should show the new kernel
   grubby --info=ALL          # Verify old kernel is still listed
   ```

4. **Schedule maintenance window:**
   - Drain traffic from the server (remove from load balancer).
   - Reboot into the new kernel.
   - Run validation tests (application health checks, network connectivity, disk mounts).

5. **Rollback plan (if issues arise):**
   ```bash
   grubby --set-default /boot/vmlinuz-<old-version>
   reboot
   ```

6. **Post-validation:** Keep the old kernel installed for at least 2 weeks before removing it with `yum remove kernel-<old-version>`.

---

**Q73. [L3] Your team is adopting SRE practices. The VP asks you to define an "Error Budget" for a critical API. What is it, and how does it influence engineering decisions?**

> *What the interviewer is testing:* SRE fundamentals, error budgets, SLO-driven development.

**Answer:**
An **Error Budget** is the maximum amount of unreliability permitted over a time window, derived directly from the SLO (Service Level Objective).

**Calculation:**
If the SLO is 99.9% availability over 30 days:
- Error Budget = 100% – 99.9% = **0.1%**
- In minutes: 30 × 24 × 60 × 0.001 = **43.2 minutes** of allowed downtime per month

**How it influences decisions:**
1. **Budget remaining (healthy):** Engineering can deploy aggressively, run experiments, ship new features. The system can tolerate some failures.
2. **Budget exhausted (unhealthy):** Feature releases are frozen. All engineering effort shifts to reliability improvements—fixing bugs, adding redundancy, improving monitoring.
3. **Shared accountability:** Product teams and SRE teams share the same budget. Product managers understand that pushing risky features consumes reliability budget.

**Practical implementation:**
- Track error budget burn rate on dashboards (Grafana/Datadog).
- Set alerts when 50%, 75%, and 90% of the budget is consumed.
- Automate deployment freezes when budget drops below a threshold.

The error budget transforms reliability from a vague goal into a measurable, actionable metric that balances innovation velocity with system stability.

---

**Q74. [L2] Your team spends 60% of its time on repetitive operational tasks (restarting services, rotating logs, provisioning VMs). In SRE terms, what is this called, and how do you reduce it?**

> *What the interviewer is testing:* Toil definition, automation strategies, SRE culture.

**Answer:**
In SRE, this is called **Toil**—manual, repetitive, automatable, reactive work that scales linearly with service size and adds no enduring value.

Google's SRE book recommends keeping toil below **50%** of an SRE team's time. The remaining 50%+ should be spent on engineering work that reduces future toil.

Strategies to reduce toil:
1. **Automate repetitive tasks:**
   - Service restarts → Systemd auto-restart with `Restart=on-failure`
   - Log rotation → Properly configured `logrotate` or journald limits
   - VM provisioning → Terraform/Ansible Infrastructure as Code

2. **Self-healing systems:**
   - Health check endpoints with automatic remediation
   - Kubernetes liveness/readiness probes that auto-restart unhealthy pods
   - Auto-scaling groups that replace failed instances

3. **Reduce ticket-driven work:**
   - Self-service portals for common requests (database access, environment creation)
   - ChatOps bots for routine operations

4. **Measure and track:** Categorize each operational task as toil or engineering. Report toil percentage weekly. If toil exceeds 50%, escalate to management and prioritize automation projects.

The key insight: Toil isn't inherently bad—it's bad when it grows faster than the team and prevents reliability improvements.

---

**Q75. [L3] After a major outage, you're asked to lead the Post-Incident Review (also called a "Postmortem"). What is the SRE approach to postmortems, and what must the document contain?**

> *What the interviewer is testing:* Blameless postmortem culture, incident learning, documentation standards.

**Answer:**
SRE postmortems are **blameless**—they focus on systemic failures, not individual mistakes. The goal is learning and preventing recurrence, not assigning blame.

A postmortem document must contain:
1. **Title and severity:** Clear description and impact level (e.g., "P1: Payment API 45-minute outage").
2. **Summary:** One-paragraph overview of what happened.
3. **Impact:** Quantified damage—users affected, revenue lost, SLO budget consumed, duration.
4. **Timeline:** Minute-by-minute chronology from detection to resolution:
   ```
   14:02 — Monitoring alert fires for 5xx rate > 5%
   14:05 — On-call acknowledges, begins investigation
   14:12 — Root cause identified: bad config deployed
   14:15 — Config rollback initiated
   14:22 — Service fully recovered
   ```
5. **Root cause analysis:** The "5 Whys" method or fault tree analysis to find the true systemic root cause—not "engineer made a mistake," but "the deployment pipeline lacked config validation."
6. **Contributing factors:** What made detection slow? What made recovery harder?
7. **Action items:** Each must have an owner, priority, and deadline:
   - `[P0] Add config validation to CI pipeline — @alice — 2 weeks`
   - `[P1] Improve alerting threshold — @bob — 1 week`
8. **Lessons learned:** What went well, what went poorly.

The postmortem is shared openly across the organization. Recurring themes across postmortems indicate systemic reliability gaps.

---

**Q76. [L2] Your monitoring dashboard shows that a server's memory usage has been steadily climbing 1% per day for two weeks. The application developers say "it's fine." How do you approach this as an SRE?**

> *What the interviewer is testing:* Capacity planning, trend analysis, proactive SRE.

**Answer:**
A steady, linear increase in memory usage is a classic **memory leak** pattern. Even if the application functions today, at 1%/day, the server will OOM in approximately 2–3 weeks.

SRE approach:
1. **Quantify the risk:**
   - Current usage: 70% → OOM in ~30 days
   - Calculate the exact date the server will hit critical threshold (e.g., 95%)
   - Present a clear timeline to developers: "This server will crash on [date] without intervention"

2. **Gather evidence:**
   ```bash
   # Track per-process memory growth
   ps aux --sort=-%mem | head -20
   # Check for growing heap
   pmap -x <PID> | tail -1
   # Monitor over time
   pidstat -r -p <PID> 60
   ```

3. **Identify the leaking process:** Compare RSS (Resident Set Size) growth across processes. The one growing linearly is the culprit.

4. **Short-term mitigation:**
   - Schedule periodic application restarts during off-peak hours
   - Set up memory-based alerts at 85% and 95%
   - Implement cgroup memory limits to prevent one process from OOMing the entire server

5. **Long-term fix:** Work with developers to profile the application using language-specific tools (Valgrind for C/C++, heap dumps for Java, memory profilers for Python/Go).

Never dismiss a linear trend—it's a countdown timer to an outage.

---

**Q77. [L3] Your company's SLO for an API is 99.95% availability. The CTO asks, "Why not just set it to 99.99%?" What is the SRE argument against over-committing to higher SLOs?**

> *What the interviewer is testing:* SLO design philosophy, cost-reliability tradeoff, diminishing returns.

**Answer:**
Higher SLOs are exponentially more expensive to achieve, and over-committing creates perverse incentives.

**The math:**
| SLO | Allowed downtime/month | Allowed downtime/year |
|-----|----------------------|---------------------|
| 99.9% | 43.8 min | 8.77 hours |
| 99.95% | 21.9 min | 4.38 hours |
| 99.99% | 4.3 min | 52.6 min |

Going from 99.95% to 99.99% means reducing allowed downtime from 21.9 minutes to 4.3 minutes—a **5x reduction**. This requires:
1. **Redundant everything:** Multi-region active-active, automated failover, zero-downtime deployments
2. **Zero human-in-the-loop:** 4.3 minutes is too short for humans to detect, diagnose, and fix—everything must be automated
3. **Dependency chain ceiling:** If your database (e.g., RDS) offers 99.95% SLA, your service physically cannot achieve 99.99% unless you build multi-database redundancy

**Perverse incentives of over-committing:**
- Development velocity plummets (every change is a risk to the SLO)
- Teams become risk-averse and stop innovating
- Error budget is perpetually exhausted, causing permanent deployment freezes

**SRE recommendation:** Set the SLO at the level users actually need. If users tolerate 1-minute blips without complaints, 99.99% wastes engineering effort that could improve features. Match the SLO to user expectations and business requirements, not aspirational perfection.

---

**Q78. [L2] You have been given a runbook for handling a "database replication lag" incident, but it's out of date. What should a well-structured SRE runbook contain?**

> *What the interviewer is testing:* Operational documentation, runbook standards, incident response.

**Answer:**
A well-structured runbook is the difference between a 5-minute resolution and a 2-hour scramble. It should be written for the worst case: an on-call engineer at 3 AM who has never seen this specific failure before.

Essential sections:
1. **Title and ownership:** "Database Replication Lag Remediation" — Owner: @database-team, Last updated: [date]

2. **Overview:** One paragraph describing what this runbook addresses and when to use it.

3. **Alert trigger:** Which monitoring alert triggers this runbook (link to alert definition).

4. **Impact assessment:**
   - What breaks if replication lag exceeds 30s? 5min? 30min?
   - Which services are affected?
   - Customer-facing impact description

5. **Diagnostic steps (copy-paste commands):**
   ```bash
   # Check replication lag
   mysql -e "SHOW SLAVE STATUS\G" | grep Seconds_Behind_Master
   # Check for long-running queries on primary
   mysql -e "SHOW PROCESSLIST" | grep -v Sleep
   ```

6. **Remediation steps:** Ordered from least risky to most risky:
   - Step 1: Kill long-running queries on primary
   - Step 2: Increase replication parallel workers
   - Step 3: Failover to standby (with explicit commands)

7. **Escalation path:** Who to call if the runbook doesn't resolve the issue, with contact details.

8. **Rollback instructions:** How to undo each remediation step if it makes things worse.

---

**Q79. [L3] Explain the concept of Chaos Engineering and how you would implement a basic chaos experiment on a Linux production system.**

> *What the interviewer is testing:* Chaos engineering principles, controlled failure injection, resilience validation.

**Answer:**
**Chaos Engineering** is the discipline of proactively injecting controlled failures into production systems to discover weaknesses before they cause real outages.

Core principles (from Netflix's Chaos Engineering):
1. **Start with a hypothesis:** "If one app server fails, the load balancer will route traffic to healthy servers within 10 seconds."
2. **Minimize blast radius:** Start with non-critical environments and gradually move to production.
3. **Run experiments during business hours:** When engineers are available to respond.
4. **Automate rollback:** Every experiment must have an automatic abort condition.

Basic chaos experiments on Linux:
1. **CPU stress test:**
   ```bash
   stress-ng --cpu 4 --timeout 300s
   ```
   Validates: Does auto-scaling trigger? Do health checks detect degraded performance?

2. **Network latency injection:**
   ```bash
   tc qdisc add dev eth0 root netem delay 200ms 50ms
   # Rollback: tc qdisc del dev eth0 root
   ```
   Validates: Do circuit breakers activate? Do timeouts work correctly?

3. **Process kill:**
   ```bash
   kill -9 $(pgrep myapp)
   ```
   Validates: Does systemd restart the process? Does the load balancer detect the failure?

4. **Disk fill:**
   ```bash
   fallocate -l 10G /tmp/fill_disk
   # Rollback: rm /tmp/fill_disk
   ```
   Validates: Do disk space alerts fire? Does the application handle disk-full errors gracefully?

Tools like **Chaos Monkey**, **Litmus Chaos**, and **Gremlin** automate these experiments at scale with safety controls.

---

**Q80. [L2] You are setting up an on-call rotation for your SRE team. What are the best practices for a healthy and sustainable on-call process?**

> *What the interviewer is testing:* On-call practices, alert fatigue, SRE team health.

**Answer:**
On-call is a critical SRE function, but poorly managed on-call leads to burnout, high turnover, and slower incident response.

Best practices:
1. **Rotation structure:**
   - Minimum 2-person rotation (primary + secondary) to prevent single points of failure
   - 1-week rotations (longer leads to burnout; shorter disrupts sleep patterns)
   - Follow-the-sun model for global teams (no one is on-call during sleep hours)

2. **Alert quality:**
   - Every alert must be **actionable**—if the on-call can't do anything about it, it shouldn't page
   - Target: fewer than **2 pages per 12-hour shift** on average
   - Review and eliminate noisy alerts monthly (alert fatigue kills response time)
   - Use severity levels: P1 (page immediately), P2 (page during business hours), P3 (ticket only)

3. **Documentation and tooling:**
   - Every paging alert must link to a runbook
   - On-call engineers must have pre-provisioned access to all production systems
   - Incident communication channels (Slack, PagerDuty, status page) must be set up

4. **Compensation and balance:**
   - On-call engineers receive compensation (extra pay, time off in lieu)
   - After being paged overnight, the engineer gets the next day off or a late start
   - Track on-call load per engineer to ensure equitable distribution

5. **Continuous improvement:**
   - Hold weekly on-call handoff meetings: outgoing on-call briefs incoming on-call
   - Review every page: Was it actionable? Was the runbook sufficient? What can be automated?
   - Track metrics: pages per shift, time-to-acknowledge, time-to-resolve
