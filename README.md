# Retail Store Operating System

A client-server system for managing retail store operations, built as an OS course project. It demonstrates multithreading, IPC via Unix signals, HTTP communication, and SQLite database integration.

## Overview

The system consists of three main components:

| Component | Language | Role |
|-----------|----------|------|
| Server | C | HTTP server handling all business logic and database access |
| Client | Bash | Interactive menu-driven frontend |
| Controller | C | Process control via Unix signals |

---

## Architecture

```
Client (Bash)
    │  HTTP requests (curl → port 8880)
    ▼
Server (C) ── Request Queue (shared memory) ── Worker Threads
    │
    ▼
SQLite Database
```

The server uses a producer-consumer model: incoming HTTP requests are enqueued into a shared-memory request queue, and worker threads dequeue and process them concurrently. A mutex and condition variable synchronize access to the queue.

---

## Components

### Server (`server/`)

- Built with **libmicrohttpd** for HTTP handling
- Uses **SQLite** for persistent storage (`server/Database/Database.db`)
- Uses **cJSON** for JSON parsing/serialization
- Multithreaded with **pthreads** — queue protected by mutex + condition variable
- Listens on **port 8880**

**API Endpoints:**

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/view_inventory` | Returns all inventory items |
| POST | `/update_inventory` | Adds/updates an item's stock |
| POST | `/process_orders` | Processes a client order |
| GET | `/generate_report` | Generates a sales report |

**Request payloads:**

```json
// POST /update_inventory
{ "Item": "Apple", "Amount": 50 }

// POST /process_orders
{ "Client": "John", "Items": [{ "Item": "Apple", "Value": 3 }] }
```

**Build & run:**
```bash
cd server
bash server.sh
```

The script compiles `main.c` and starts the server:
```bash
gcc -o server main.c -lmicrohttpd -lcjson -lpthread -lsqlite3
./server
```

---

### Client (`client/client.sh`)

An interactive Bash script that presents a menu and communicates with the server via `curl`.

**Menu options:**
1. **View Inventory** — GET `/view_inventory`
2. **Update Inventory** — POST `/update_inventory` with item name and amount
3. **Process Order** — POST `/process_orders` with client name and item list
4. **Customized Commands** — sub-menu for `generate_report` and `search_inventory`
5. **Exit**

**Run:**
```bash
bash client/client.sh
```

---

### Controller (`controller/`)

A C program that sends Unix signals to running server processes, enabling runtime control without restarting.

**Usage:**
```bash
./controller <server_pid1> [<server_pid2> ...]
```

**Signal options:**

| Option | Signal | Effect |
|--------|--------|--------|
| 1 | SIGTERM | Graceful termination |
| 2 | SIGSTOP | Pause server |
| 3 | SIGCONT | Resume server |
| 4 | SIGKILL | Force kill |

If a PID is stale, the controller automatically refreshes the PID list using `pgrep`.

**Build:**
```bash
cd controller
gcc -o controller main.c
```

---

### Custom Commands (`commands/`)

Standalone Bash scripts installed at `/usr/local/bin/` and accessible from the client's Customized Commands menu.

- **`generate_report.sh`** — Fetches a full sales report from the server
- **`search_inventory.sh <keyword>`** — Fetches inventory and filters results by keyword using `grep`

---

## Dependencies

- **gcc** — C compiler
- **libmicrohttpd** — Embedded HTTP server library
- **cJSON** — Lightweight JSON parser for C
- **sqlite3** — Embedded relational database
- **pthreads** — POSIX threads (usually bundled with glibc)
- **curl** — Used by client and command scripts to send HTTP requests

Install on Debian/Ubuntu:
```bash
sudo apt install gcc libmicrohttpd-dev libcjson-dev libsqlite3-dev curl
```

---

## Project Structure

```
Server_Project/
├── server/
│   ├── main.c              # HTTP server with request queue and worker threads
│   ├── server.sh           # Build and run script
│   ├── Headers/
│   │   ├── View_Inventory.h
│   │   ├── Update_Inventory.h
│   │   ├── Process_Orders.h
│   │   └── Generate_Report.h
│   └── Database/
│       └── Database.db     # SQLite database
├── client/
│   └── client.sh           # Interactive menu client
├── controller/
│   ├── main.c              # Signal-based process controller
│   └── controller.sh       # Helper script
└── commands/
    ├── generate_report.sh  # Custom command: fetch sales report
    └── search_inventory.sh # Custom command: search inventory by keyword
```
