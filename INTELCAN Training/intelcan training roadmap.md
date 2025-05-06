# Intelcan Linux Roadmap (CLI + AWS Edition) — **Detailed Guide**
---

## How to use this roadmap
* Each phase is **two weeks of evening work** (≈ 5 × 2‑hour sessions).  
* All drills are **terminal‑only**; no GUI is required (and usually impossible on a t2.micro).  
* The left‑hand bullet explains the **concept**; the right‑hand code‑block is a **hands‑on task** so you can discover details yourself.  
* “Self‑check” boxes are tests you can run to know you grasped the idea without me giving away answers.  
* Internal links (`[[like this]]`) turn into backlinks in Obsidian so you can grow your own knowledge base.

---

# Phase 0 – AWS Free‑Tier Setup
[[Roadmap Index]]

### Concepts You’ll Meet
| Concept | Why it matters |
|---------|----------------|
| **IAM & Key Pairs** | All AWS CLI calls sign requests with an Access & Secret key. You’ll learn least‑privilege by granting only the APIs you need. |
| **Security Groups (SG)** | AWS’s built‑in state‑ful firewall; you must open only TCP 22 to *your* IP. |
| **AMI & EBS Snapshot** | Equivalent to VM templates & disk checkpoints—fast rollback for experiments. |
| **Package Groups** | `dnf groupinstall "Development Tools"` pulls a proven compiler tool‑chain without hunting packages one‑by‑one. |

> See also: [[Concepts/Security Groups]], [[Concepts/IAM Roles]]

### Step‑by‑Step

1. **Create the instance**  
   ```bash
   aws ec2 run-instances         --image-id ami-al2023-latest         --instance-type t2.micro         --key-name mykey         --security-group-ids sg-CLI         --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=sky_lab}]'
   ```
   *Self‑check:* `ssh -i mykey.pem ec2-user@<ip>` drops you in a shell.

2. **Lock down the SG** – *principle of least privilege*  
   ```bash
   MY_IP=$(curl -s https://checkip.amazonaws.com)/32
   aws ec2 authorize-security-group-ingress         --group-id sg-CLI         --ip-permissions IpProtocol=tcp,FromPort=22,ToPort=22,IpRanges=[{CidrIp=$MY_IP,Description="SSH from home"}]
   ```
   *Why:* default rule `0.0.0.0/0` invites brute‑force bots. citeturn5file1

3. **Install baseline tool‑chain**  
   ```bash
   sudo dnf groupinstall "Development Tools" -y
   sudo dnf install clang lldb cmake git tmux htop python3-tshark -y
   ```
   *Self‑check:* `gcc --version` shows ≥ 11; `tshark -D` lists NICs. citeturn5file0

4. **Snapshot before you break things**  
   ```bash
   aws ec2 create-image --instance-id $(curl -s http://169.254.169.254/latest/meta-data/instance-id)         --name baseline-$(date +%F)
   ```
   *Self‑check:* New AMI ID appears in `aws ec2 describe-images --owners self`.

5. **(Optional) Cost alarms** – stay free  
   ```bash
   aws budgets create-budget         --account-id <12‑digit‑id>         --budget file://monthly_0.50.json
   ```
   Where `monthly_0.50.json` caps EC2 to \$0.50/month.

### Expected Outcome
* One **t2.micro** instance reachable via SSH only from your IP.  
* `gcc`, `clang`, `tmux`, `tshark` all runnable.  
* AMI snapshot named `baseline‑YYYY‑MM‑DD`.

---

# Phase 1 –[[intelcan_cli_aws_roadmap_detailed]] Daily Linux Survival (Weeks 1–2)
[[Roadmap Index]]

### Big Ideas
* **Shell navigation** – everything is a file or a process.  
* **POSIX permissions vs. ACLs** – controlling access without chmod chaos.  
* **Process scheduling** – `nice`, `taskset`, real‑time priorities.  
* **Package life cycle** – why distros prefer RPM/DEB over “make install”.  

### Week 1 Checklist

| Day | Concept | Drill | Self‑check |
|-----|---------|-------|------------|
| 1 | *TTY vs. PTY* | Switch to a raw TTY on your local machine (`Ctrl+Alt+F3`) and SSH into EC2 for one hour. | Did you avoid using a GUI terminal emulator? |
| 2 | *Filesystem & Logs* | `cd /var/log && sudo less secure` – trace an auth failure. | Can you explain each field in `/var/log/secure`? |
| 3 | *Permissions* | ```bash
sudo mkdir -p /var/log/sky_demo
sudo chown skyuser:skyuser /var/log/sky_demo
sudo chmod 750 /var/log/sky_demo
``` | `namei -l /var/log/sky_demo` shows correct owner/mode. citeturn5file2 |
| 4 | *Process control* | ```bash
yes > /dev/null & PID=$!
sudo renice -n -10 $PID
sudo taskset -c 2 $PID
``` | `ps -o pid,ni,psr,comm -p $PID` reflects new values. |
| 5 | *Reflection* | Run `script week1.cast`; archive every command. | Can you grep mistakes in your own history? |

### Week 2 – Automate & Package
* Turn the Week 1 tasks into Bash scripts in `~/bin`.  
* Build **htop** from source and package it with `checkinstall` to produce an RPM.  
  *Why:* identical to the exercise in the original roadmap citeturn5file4.  
* Push your `~/bin` scripts and the RPM to a private Git repo.

### Upgrade Watch
If `make -j$(nproc)` takes >10 min, stop the instance and launch **t4g.micro** (still free but Arm64). For x86_64 keep tasks short.

---

# Phase 2 – Coding on Linux (Weeks 3–4)
[[Roadmap Index]]

### Concepts
* **Out‑of‑tree builds** – isolate source & object dirs with CMake.  
* **Static analysis** – `clang‑tidy` enforces coding style before CI.  
* **Debug & Profiling** – `gdb`, `valgrind`, `perf`.  

### Tasks

1. **Scaffold a C++20 project**  
```bash
   mkdir ~/code/radar_demo && cd $_
   cat > CMakeLists.txt <<'EOF'
   cmake_minimum_required(VERSION 3.18)
   project(radar_demo LANGUAGES CXX)
   set(CMAKE_CXX_STANDARD 20)
   add_executable(radar_demo main.cpp)
   EOF
   echo '#include <iostream>
int main(){std::cout<<"demo\n";}' > main.cpp
   cmake -B build && cmake --build build -j
```
2. **Add clang‑tidy gate** – fail build on warning (research `CMAKE_CXX_CLANG_TIDY`).  
3. **Introduce a null‑ptr bug** then trace it in gdb & valgrind.  
4. **Package** with `cpack -G RPM`.

*Upgrade trigger:* parallel compile or valgrind may need **t3.small (2 vCPU)**. Start/stop to avoid charges.

---

# Phase 3 – System‑side Linux (Weeks 5–6)
[[Roadmap Index]]

### Concepts
* **systemd** units & watchdogs.  
* **Network stack** – `ip`, `ss`, UDP multicast.  
* **SELinux** – policy modules, never disable.  

### Labs

| Topic | Hands‑on |
|-------|----------|
| `systemd` | Craft `sky_demo.service` with `Restart=on-failure`. Enable & observe `journalctl -u sky_demo -f`. citeturn5file2 |
| Packet crafting | Use **Scapy** to emit an ASTERIX Cat 048 UDP datagram, capture with `tcpdump -ni eth0 udp port 30002 -vv`. citeturn5file2 |
| Time sync | Configure `chronyd` as NTP client; parse `chronyc tracking`. |
| SELinux | Trigger an AVC by binding port 600; build a minimal module with `audit2allow -M` and load with `semodule -i`. |

*Upgrade trigger:* Simulating two nodes? Spin a second **t2.micro** and place both in a VPC subnet.

---

# Phase 4 – Real‑time & High‑Availability (Weeks 7–8)
[[Roadmap Index]]

### Concepts
* **PREEMPT_RT** – Linux with deterministic latency.  
* **Pacemaker & Corosync** – quorum & fail‑over.  
* **Cyclictest** – measure scheduling jitter.

### Exercises

1. **Kernel build** – Rebuild Rocky’s RT kernel SRC RPM, install, set default with `grubby`, verify with `uname -a` shows `PREEMPT_RT`. citeturn5file2  
2. **Latency benchmark** – `cyclictest -t4 -p99 -n -i200 -d0 -l100000`; aim for <50 µs max.  
3. **Two‑node HA** – bring up `pcs` cluster, add `sky_demo` as a managed resource, pull the primary node’s plug. citeturn5file7

*Upgrade trigger:* Kernel compile requires **t3.medium (2 vCPU / 4 GiB)**. Stop afterwards.

---

# Phase 5 – Domain Demo (Weeks 9–10)
[[Roadmap Index]]

### Concepts
* **ADS‑B & ASTERIX** – aviation surveillance data models.  
* **Headless CI** – Jenkins in Docker, QEMU smoke‑test.  

### Build‑it Project

1. **ADS‑B to ASTERIX converter**  
   * Parse dump1090 JSON/TCP feed, emit Cat 062 UDP.  
2. **(Optional)** Headless radar PPI via ASCII art; skip Qt for now.  
3. **CI pipeline**  
   * Jenkins stages: *Build → UnitTests → Package → QEMU smoke‑test*. citeturn5file7

*Upgrade trigger:* Jenkins + QEMU spins want **t3.large** but only while the job runs.

---

# Upgrade Guidance
[[Roadmap Index]]

| Phase | Task | Temporary Instance | Why / Cost |
|-------|------|--------------------|------------|
| 2 | Large parallel compile | t3.small | 2 vCPU, \$0.018/hr |
| 4 | PREEMPT_RT build | t3.medium | 4 GiB RAM prevents OOM, \$0.037/hr |
| 4 | 2‑node cluster | +1 t2.micro | Free tier hours permitting |
| 5 | Jenkins + QEMU | t3.large | Needs 8 GiB; stop after pipeline |

---

# Concepts Index
[[Roadmap Index]]

* [[Concepts/Security Groups]] — inbound vs. egress, stateful rules.  
* [[Concepts/IAM Roles]] — temporary credentials vs. long‑lived keys.  
* [[Concepts/PREEMPT_RT]] — how Linux gains real‑time determinism.  
* [[Concepts/ASTERIX]] — Eurocontrol surveillance standard.

(Feel free to create each concept as its own note and backlink.)

---

*End of file — grow it, fork it, make it yours.*