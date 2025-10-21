+++
author = "Authors"
title = "How to troubleshoot when encounter High NI cpu usage"
date = "2025-10-09"
description = "Troubleshoot NI CPU usage"
tags = [
    "NI",
    "cpu",
]
+++


Not necessarily 🙂 — adding more CPU cores is **one solution**, but it’s not the **only way**.
What you’re seeing (`62.2% ni + 24.5% sy`) means **one process with a nice priority is dominating CPU**, and the kernel is busy servicing it.

Here are options you can consider **before throwing hardware at it**:

---

## 🔹 1. Tune process priority & scheduling

* Keep the workload at a **lower priority (`nice +10` or more)** if it’s background/batch work → that way it won’t starve other system tasks.
* If it’s critical, you can give it higher priority (`renice 0` or negative), but this can **hurt system responsiveness**.

---

## 🔹 2. Limit CPU usage (containment)

Instead of letting it hog the CPU:

* **cpulimit**:

  ```bash
  cpulimit -p <PID> -l 50
  ```

  → caps the process to 50% of a core.
* **cgroups / systemd**:

  ```bash
  systemctl set-property myservice.service CPUQuota=50%
  ```
* **Docker/Kubernetes**:
  Use `--cpus` or `cpu_shares` to limit.

This way you balance the load across processes.

---

## 🔹 3. Optimize the workload

* Check **why** the process is eating so much CPU:

  * Is it in a **tight loop**?
  * Using inefficient syscalls?
  * Could the algorithm be optimized?
* Use **`perf top`**, **`strace -p <PID>`**, or **`pidstat`** to see what it’s doing.
* Sometimes small code/config tweaks reduce CPU massively.

---

## 🔹 4. Spread the load

* If it’s parallelizable, run it across multiple cores/servers.
* Tools: job schedulers, containers, or load balancers.

---

## 🔹 5. Increase CPU resources

If the workload is truly **legitimately CPU-bound** (e.g., video encoding, scientific computing, crypto), and it’s already optimized, then yes:

* Add **more vCPUs** (in a VM).
* Add **more cores/servers** (scale out).

---

✅ **So:** Increasing CPU resources is often the *last resort*. First, try:

* **Renice or limit CPU** (contain it).
* **Investigate what it’s doing** (optimize).
* **Distribute workload** (scale out).

---

**practical checklist** 

---

# 📝 CPU Usage Troubleshooting Checklist

---

## 🔹 Step 1: Confirm which process is eating CPU

```bash
top -o %CPU
```

* Look at the **PID**, **%CPU**, and **NI (nice value)**.
* Note if it’s a single process dominating or multiple ones.

Alternative:

```bash
ps -eo pid,ppid,ni,pri,pcpu,comm --sort=-pcpu | head -20
```

---

## 🔹 Step 2: Check if it’s kernel-heavy or user-heavy

Use `pidstat`:

```bash
pidstat -u 1
```

* `%usr` → user code.
* `%system` → kernel/syscalls.
* If `%system` is high, the process may be hammering I/O, networking, or syscalls.

---

## 🔹 Step 3: Investigate what it’s doing

1. **Trace syscalls**:

   ```bash
   strace -p <PID>
   ```

   → shows system calls in real-time (e.g., excessive `read()`, `write()`, `futex()` loops).

2. **Profile hotspots**:

   ```bash
   perf top
   ```

   → shows where CPU time is being burned (user code vs kernel).

---

## 🔹 Step 4: Control its resource usage

* **Renice** if it’s background work:

  ```bash
  sudo renice -n 10 -p <PID>
  ```
* **Limit CPU**:

  ```bash
  cpulimit -p <PID> -l 50
  ```
* **Systemd cgroup limit**:

  ```bash
  systemctl set-property myservice.service CPUQuota=50%
  ```
* **Docker/K8s**:

  ```bash
  docker run --cpus="1.0" ...
  ```

---

## 🔹 Step 5: Optimize or scale

* If it’s inefficient code → optimize algorithms.
* If it’s parallelizable → spread load across cores/nodes.
* If it’s **legitimately CPU-heavy** and already optimized → add more vCPUs or nodes.

---

## ✅ Decision Matrix

* **One background process eating CPU →** `renice` or `cpulimit`.
* **Kernel/system CPU high →** investigate with `perf`/`strace`.
* **Repeated heavy workload (e.g. encoding, analytics) →** optimize code or parallelize.
* **Already optimized and still maxed out →** increase CPU resources.

---



