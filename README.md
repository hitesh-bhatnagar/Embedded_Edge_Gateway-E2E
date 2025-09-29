# Embedded_Edge_Gateway-E2E

Excellent — love this. We’ll build a **single, deployment-ready project** that *demonstrates every major Embedded Linux / System Programming skill you’ve learned so far*, and that you can point to on your CV + demo in interviews.

I’ll give you:

1. A short project idea & why it’s perfect for placements.
2. A complete **file/folder layout**.
3. A module-by-module plan (what each file does, which syscalls/APIs it demonstrates, and why).
4. A sample **Makefile**, build & run commands.
5. Deployment & demo: systemd service, startup scripts, tests.
6. What to put in README / resume + suggested interview talking points.
7. A simple test checklist and next-extension ideas.

Follow this step-by-step and you’ll have a polished project that *clearly* maps to interview topics.

---

# 1) Project idea — **Embedded Edge Gateway (E2E demo)**

**Name:** edge-gateway
**Short description:** a local gateway that simulates multiple sensors, aggregates sensor data, processes it, and exposes it to clients.
**Why this is ideal:** it uses processes, threads, pipes, message queues, shared memory + semaphores, sockets (TCP & UDP), signals, file I/O, Makefiles, shell scripts, and simple deployment — everything an interviewer can ask.

**High-level features**

* A **parent manager** process launches:

  * multiple **sensor processes** (simulated sensors using fork/exec) that write into shared memory (or push via message queue),
  * one **aggregator/logger** process that reads shared memory and writes to logs (synchronized by semaphores),
  * a **monitor thread** inside the parent that reports CPU/health (pthread),
  * a **TCP server** (inside parent or separate process) that provides a JSON endpoint for clients to request the latest aggregated data,
  * a **UDP broadcaster** to broadcast heartbeat/status for discovery.
* A **controller CLI or client** that connects (TCP) and requests data.
* A **shell script** to build/start/stop and a **systemd** unit to deploy on a Linux machine (Raspberry Pi later).
* A simple **automated test script** that runs clients and asserts expected behavior.

This project gives you full-stack system programming coverage: processes, exec, wait, IPC, threads, sockets, file I/O, scripts, deployment.

---

# 2) File / folder layout (recommended)

```
edge-gateway/
├─ include/
│   ├─ common.h          # shared types, keys, macros
│   └─ ipc.h             # wrappers for shm/msg/sem
├─ src/
│   ├─ manager.c         # parent manager: starts components, monitor thread, TCP server
│   ├─ sensor.c          # sensor executable (fork/exec'ed)
│   ├─ aggregator.c      # aggregator/logger process (reads IPC -> write logs)
│   ├─ ipc_helpers.c     # implementation for msgqueue/shm/sem helpers
│   └─ tcp_server.c      # optional separate TCP server (or integrated in manager)
├─ scripts/
│   ├─ start.sh
│   ├─ stop.sh
│   └─ test.sh
├─ systemd/
│   └─ edge-gateway.service
├─ Makefile
├─ README.md
└─ demo/                 # example client/tools (python client) and sample logs
    ├─ client.py
    └─ read_logs.sh
```

---

# 3) Module-by-module plan (what to code + why — interview mapping)

### `include/common.h`

* Holds constants and IPC keys (ftok style), `shm_payload_t`, `command_t`.
* Why: centralized types; shows you design modular C projects.

Key items:

```c
typedef struct {
    unsigned int seq;
    int temperature;
    int humidity;
    time_t ts;
} shm_payload_t;

typedef enum { CMD_START=1, CMD_STOP=2, CMD_EXIT=3 } command_t;
```

### `include/ipc.h` + `src/ipc_helpers.c`

* Implement small wrappers:

  * msg queue: `mq_create(key)`, `mq_send()`, `mq_recv()`
  * shm: `shm_create()`, `shm_attach()`, `shm_detach()`, `shm_remove()`
  * sem: `sem_create()`, `sem_setval()`, `sem_wait_idx()`, `sem_post_idx()`, `sem_remove()`
* **Why:** This isolates SysV complexity; shows you know System V IPC and why wrappers are useful.

Interview notes: explain `ftok`, differences between System V vs POSIX IPC, and when to choose msgq vs shm.

### `src/sensor.c` (executable)

* Behavior:

  * Reads configuration (id, sample interval) from argv or env.
  * Writes generated sensor sample into shared memory or sends via message queue to aggregator.
  * Uses `sleep()`/`usleep()` to simulate periodic sampling.
  * Handles `SIGTERM` to exit gracefully.
* Syscalls/APIs used:

  * `getpid()`, `fork()` not here (manager execs it), `shmat`/`msgsnd`/`semop` for access, `signal()`.
* Why: demonstrates sensor logic separated into a process and shows `exec` use from manager.

### `src/aggregator.c`

* Behavior:

  * Waits for new data (either via sem + shared memory or via msg queue).
  * On new sample: validates, appends to `sensor_log.csv`, updates an in-memory structure (to serve to TCP clients).
  * Can accept `CMD_EXIT` from message queue to exit.
* APIs:

  * `fopen/fprintf`, `shmat/shmdt`, `semop`, `msgrcv`, `msgsnd`, file I/O.
* Why: shows file logging, structured parsing, and synchronisation.

### `src/manager.c`

* Behavior:

  * Creates IPC (shm/msg/sem), forks/execs N sensors and aggregator, spawns monitoring thread (pthread) to periodically print CPU usage (simulated).
  * Runs an integrated TCP server or launches `tcp_server` process to accept client requests (latest aggregated values). It must handle `SIGINT` to gracefully shutdown: send `CMD_EXIT`, kill children, remove IPC resources.
* APIs:

  * `fork`, `execl`, `pthread_create`, `signal`, `kill`, `waitpid`, `socket` (if integrated).
* Why: this is the orchestrator. Demonstrates process control, signals, threads, sockets.

### `src/tcp_server.c` (optional separate process)

* A simple multi-client server using `fork()` or `pthread` per connection. On request, sends JSON of last N samples.
* APIs: `socket`, `bind`, `listen`, `accept`, `send`, `recv`, `fork` or `pthread`.
* Why: demonstrates network IPC and multi-client handling.

---

# 4) Makefile (sample)

```makefile
CC=gcc
CFLAGS=-Wall -Iinclude -O2
LDFLAGS=-pthread

SRC=src
BIN=.

SRCS=$(SRC)/manager.c $(SRC)/sensor.c $(SRC)/aggregator.c $(SRC)/ipc_helpers.c $(SRC)/tcp_server.c
OBJS=$(SRCS:.c=.o)

all: manager sensor aggregator tcp_server

manager: $(SRC)/manager.c $(SRC)/ipc_helpers.c
	$(CC) $(CFLAGS) -o $(BIN)/manager $(SRC)/manager.c $(SRC)/ipc_helpers.c $(LDFLAGS)

sensor: $(SRC)/sensor.c $(SRC)/ipc_helpers.c
	$(CC) $(CFLAGS) -o $(BIN)/sensor $(SRC)/sensor.c $(SRC)/ipc_helpers.c

aggregator: $(SRC)/aggregator.c $(SRC)/ipc_helpers.c
	$(CC) $(CFLAGS) -o $(BIN)/aggregator $(SRC)/aggregator.c $(SRC)/ipc_helpers.c

tcp_server: $(SRC)/tcp_server.c $(SRC)/ipc_helpers.c
	$(CC) $(CFLAGS) -o $(BIN)/tcp_server $(SRC)/tcp_server.c $(SRC)/ipc_helpers.c $(LDFLAGS)

clean:
	rm -f $(BIN)/manager $(BIN)/sensor $(BIN)/aggregator $(BIN)/tcp_server $(SRC)/*.o
```

* Why: teaches build automation, dependencies, and using LDFLAGS for pthread.

---

# 5) Build & run (local)

Commands:

```bash
# from project root
make
# start manager (it will spawn others)
./manager
# Or to run components manually (for debugging)
./aggregator &
./sensor 1 1000 &   # id=1, interval=1000ms
```

**Manager behavior:**

* create IPC
* `fork()` + `execl("./sensor", "./sensor", "1", "1000", NULL);` etc.
* launch aggregator similarly
* register `signal(SIGINT, handler)` to cleanup

---

# 6) Deployment & service (systemd) — how to deploy on a Linux device

### a) Create a simple start/stop script `scripts/start.sh`

```bash
#!/bin/bash
set -e
cd "$(dirname "$0")/.."
./manager >> /var/log/edge-gateway/manager.log 2>&1 &
echo $! > /var/run/edge-gateway.pid
```

`stop.sh` reads pid and kills gracefully (sends SIGINT).

### b) systemd unit `systemd/edge-gateway.service`

```ini
[Unit]
Description=Edge Gateway Service
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/edge-gateway
ExecStart=/opt/edge-gateway/manager
Restart=on-failure
User=pi
KillMode=process

[Install]
WantedBy=multi-user.target
```

* To deploy: copy binaries to `/opt/edge-gateway`, place unit file in `/etc/systemd/system/`, `systemctl daemon-reload`, `systemctl enable --now edge-gateway`.

Why systemd: shows production readiness, and interviewers like deployment knowledge.

---

# 7) Testing & demo scripts

### `scripts/test.sh`

* Starts manager (if not running), runs a Python client that requests data, asserts JSON keys exist, greps logs to ensure sensor writes, then stops the manager.

Example (pseudo):

```bash
#!/bin/bash
./manager & sleep 2
python3 demo/client.py --get-latest
grep -q "Sensor wrote" sensor_log.csv || { echo "sensor log missing"; exit 1; }
killall manager
echo "Tests passed"
```

Why: demonstrates you can automate tests.

---

# 8) README / Resume bullets / Interview talking points

### README should include:

* Project overview (one paragraph).
* Features list mapping to system concepts:

  * "Uses `fork()` / `exec()` to launch sensor processes"
  * "Inter-process communication: shared memory + System V semaphores used for fast data transfer, message queue for control messages"
  * "Aggregator logs to CSV using file I/O; TCP server exposes JSON API for clients"
  * "Manager uses pthread for monitoring and signal handling for graceful shutdown"
  * "Deployed with systemd; start/stop scripts and test automation included"
* Build & run instructions (`make`, `./manager`, start systemd)
* Directory layout and how to run the demo client.
* Short FAQ: how to extend to run on Raspberry Pi.

### Resume bullets (2–3 lines)

* "Built `edge-gateway` — an Embedded Linux edge gateway demonstrating process management (`fork/exec`), IPC (shared memory + semaphores, message queues), multithreading (`pthread`), and TCP/UDP sockets. Includes systemd deployment and test automation."
* "Implemented aggregator that writes high-throughput sensor samples to disk safely (semaphores & shared memory); implemented JSON TCP server for clients."

### Interview talking points (prep)

* Explain why shared memory + semaphores used for speed vs message queue for control (use cases).
* Explain how you handle graceful shutdown (signals -> send CMD_EXIT -> waitpid -> detach + cleanup).
* Explain race condition examples and how your code prevents them (sem_wait/sem_post or pthread_mutex).
* Explain design decisions: why separate sensor processes vs threads; pros/cons.

---

# 9) Extra polish (optional but high-impact)

* Add a `Makefile` target `make package` to create a tarball for deployment.
* Add a small web UI served by TCP server (static page + JSON API) to demo in interviews.
* Add a Dockerfile (for easy reproducible demo on laptop/dev machine).
* Add metrics or gprof sampling to show performance numbers.

---

# 10) Step-by-step implementation plan you can follow now (week-by-week)

* **Day 1:** Create header files (`common.h`, `ipc.h`) and `ipc_helpers.c` wrappers. Test shm/msg/sem helper functions with tiny test programs. (Small wins)
* **Day 2:** Implement `sensor.c` (simulate values and write to shm/msg) and run it standalone.
* **Day 3:** Implement `aggregator.c` to read shm/msg, log to CSV.
* **Day 4:** Implement `manager.c` that creates IPC, forks/execs sensor & aggregator, and adds pthread monitoring thread + SIGINT handler. Test graceful exit.
* **Day 5:** Implement `tcp_server.c` that returns last N samples via JSON. Test client.
* **Day 6:** Add scripts, Makefile, README, systemd unit.
* **Day 7:** Polish, write demo script, add README and resume bullets.

---

# 11) Quick FAQ / pitfalls & how to answer in interviews

* Q: *Why choose shared memory + semaphores?* → Answer: For high-throughput, low-latency data exchange; semaphores ensure mutual exclusion and data-consistency without heavy kernel copying.
* Q: *Why message queue vs shm?* → Control messages, commands, or small bursts are easier with message queue; structured messages and no shared memory complexity.
* Q: *How to debug IPC problems?* → Tools: `ipcs -a` to check existing System V objects, `strace` to trace syscalls, and print logs.
* Q: *How to ensure no resource leak?* → Always remove ipc objects on exit (`msgctl(..., IPC_RMID)`, `shmctl(..., IPC_RMID)`), handle signals to ensure cleanup.

---

# 12) Final: Minimal starter code pointers (what to paste first)

Start with these three simple pieces to get momentum:

1. `include/common.h` with `shm_payload_t` and keys.
2. `src/ipc_helpers.c` with one or two helper functions (`shm_create`, `shm_attach`, `sem_create`, `sem_wait/post`) and test via a tiny `test_shm.c`.
3. `src/sensor.c` — simple program to attach to shm and write values.

Build & test those first, then I’ll give you the next batch (aggregator and manager). Want me to produce those three files' full minimal implementations now so you can copy-paste and run them locally?
