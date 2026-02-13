# Copilot instructions for MTProxy

Purpose

- Short, practical guidance so an AI coding agent becomes productive immediately in this repository.

Quick start (build / run) ✅

- Build: `make` (binary: `objs/bin/mtproto-proxy`). See `Makefile` and `README.md`.
- Run (dev):
  - obtain `proxy-secret` and `proxy-multi.conf` (core.telegram.org),
  - example: `./mtproto-proxy -u nobody -p 8888 -H 443 -S <secret> --aes-pwd proxy-secret proxy-multi.conf -M 1`
- Diagnostics: `wget localhost:8888/stats` (only via loopback by default).

Big picture (high‑level architecture) 💡

- engine/ — process lifecycle, job classes, signal handling, startup (see `engine/engine.c`).
- net/ — networking primitives and event loop (epoll), connection model, HTTP RPC handlers (`net/net-events.c`, `net/net-connections.c`, `net/net-http-server.c`).
- mtproto/ — MTProto-specific logic, configuration parsing and the main proxy behaviour (`mtproto/mtproto-proxy.c`, `mtproto/mtproto-config.c`).
- jobs/ — threaded job scheduler and classes (JC_IO, JC_ENGINE, JC_CONNECTION). Use `create_async_job`, `schedule_job_callback`.
- crypto/ + net-crypto-\* — AES/DH helpers and pwd file loader (`net/net-crypto-aes.c` → `aes_load_pwd_file`).
- common/ — utility helpers (logging `kprintf`, time, parsing), config parsing lives in `common/parse-config.c`.

Core runtime patterns & conventions 🔧

- Event-driven epoll + jobs: network IO schedules jobs rather than performing heavy work on epoll thread. See `net/net-events.c` and `jobs/*`.
- Connection type pattern: define a `conn_type_t` and set `.init_accepted`, `.read_ready`, `.write_ready`. Examples: `net/net-http-server.c` (`ct_http_server`), `net/net-tcp-rpc-server.c`.
- Thread-safety assertions: call `check_thread_class(JC_*)` where required — many functions assume a job class.
- Use `CONN_INFO(c)` and `HTS_FUNC(c)` accessors for connection metadata. Do not manipulate fd/generation without these helpers.
- Logging: use `vkprintf(level, ...)` / `kprintf()`; verbosity controlled by `-v` / `--verbosity`.
- Command-line options: parsed in `engine/engine-net.c` and `common/server-functions.c` — add new CLI flags there.
- Secrets: AES secrets loaded via `aes_load_pwd_file()` called in `engine/engine_init()`; user-supplied `--aes-pwd` / `-S` options used at startup.

Where to change common behaviors (examples) 🔎

- Add new HTTP path / handler: modify `net/net-http-server.c` or register via `engine_set_http_fallback` (see `engine/engine.c`).
- Add a new connection protocol: copy an existing `conn_type_t` (see `net/*.c`) and wire it in `init_listening_connection(...)` or in `mtproto/mtproto-proxy.c`.
- Add CLI option: extend parse table in `engine/engine-net.c` or `common/server-functions.c` and handle in the appropriate `parse_option_*`.

Integration & external deps ⚙️

- Requires OpenSSL and zlib (dev headers). See README install steps.
- Runtime config: `proxy-secret` and `proxy-multi.conf` come from core.telegram.org; AES pwd file consumed at startup.
- Optional: systemd example in README; official (but outdated) Docker image exists.

Observability & debugging tips 🐞

- Increase verbosity: `-v` (or `--verbosity`). Many routines guard extra logs with `verbosity` checks.
- Use local HTTP stat port (`-p <local-port>`) and query `/stats`.
- Fail-fast asserts are common — respect `check_thread_class()` and `assert()` hints.
- For reproducing race/thread issues, run with `-M 1` (single worker) to simplify behavior.

Files to inspect first (fast map) 📂

- `mtproto/mtproto-proxy.c` — main proxy logic, RPC + HTTP fallback handling
- `engine/engine.c` — startup, job class initialization
- `net/net-events.c` — epoll + event heap
- `net/net-connections.c` — connection lifecycle, `conn_type_t` usage
- `net/net-http-server.c` — HTTP endpoints and helpers (use this for `/stats` changes)
- `net/net-crypto-aes.c` — secret file loading (`aes_load_pwd_file`)
- `common/parse-config.c` — config-file parsing helpers
- `jobs/jobs.h` — job classes and macros (JC_IO, JC_ENGINE, etc.)

Do/Don't (project‑specific) ✅/⚠️

- ✅ Use existing `conn_type_t` + job scheduling patterns when adding protocols.
- ✅ Respect `check_thread_class()` — many data-structures are class‑guarded.
- ⚠️ Don’t add global locks without strong reason — code relies on per-job-class threading.
- ⚠️ Don’t change packet parsing conventions silently — mtproto formats and RPC timeouts are sensitive.

Examples (concrete references)

- To load AES secrets at startup: see `engine/engine.c` → `aes_load_pwd_file(pwd_filename)`.
- HTTP server skeleton: `net/net-http-server.c` defines `ct_http_server` and `default_http_server`.
- Job creation: search `create_async_job(...)` (used widely in `mtproto/mtproto-proxy.c` and `engine/*`).

If anything above is unclear or you want more detail on a specific area (e.g. add-new-protocol, add-CLI-flag, or tracing a runtime bug), tell me which task and I'll expand this doc or add a short how‑to. ✨
