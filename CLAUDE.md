# Open Gadget

Open-source, low-latency controller for industrial sensors: sensors and model
pipelines run as separate containers and exchange ZeroMQ messages so inspection
runs event-driven and in parallel. Targets x86 and ARM Linux.
Repo: https://github.com/caaden/OpenGadget

Design lineage: LMI's Gadget_Inspection_Template (see `docs/adr/0001-origin.md`).
Open Gadget is a clean implementation of that architecture, not a fork.

<!-- Maintainer notes in HTML comments are stripped before Claude reads this file. -->

## Commands

- Install: `uv sync`
- Test: `uv run pytest`
- Lint + format: `uv run ruff check . && uv run ruff format .`
- Type check: `uv run mypy src`
- Full gate (run before claiming any task done): `make check`
- Run the simulated stack: `docker compose -f compose.sim.yaml up`

<!-- Decide: mypy vs pyright. Replace the type-check line once chosen. -->

## Layout

- `src/opengadget/` — the controller: one package per service
  (`sensor/`, `pipeline/`, `automation/`, `hmi/`, `datamanager/`, `cloud/`)
- `src/opengadget/contracts/` — message schemas and service lifecycle states
- `apps/<name>/` — user applications; the reference app is `apps/reference/`
- `sim/` — sensor simulator and soft PLC used by tests and `compose.sim.yaml`
- `docs/adr/` — architecture decision records (`NNNN-title.md`)
- `tests/` — mirrors `src/`; integration tests under `tests/integration/`

## The user contract (do not break)

A user configures a new application with exactly these four things and
nothing else:

1. `pipeline.py` — receives sensor frames, runs the model pipeline, posts results
2. `automation.py` — receives results, writes to the PLC
3. an HMI config file
4. `Dockerfile`s, `docker-compose.yaml`, `.env`

Users never modify controller code. If a task seems to require a user to
touch anything outside those four files, stop and raise it instead.

## Protected files

Changing any of these requires an ADR in the same change and explicit
approval from the maintainer. Propose the change in plan mode first.

- `src/opengadget/contracts/**`
- `apps/reference/**`
- `compose.sim.yaml`
- `docs/adr/**`

## Conventions

- Python 3.12, `from __future__ import annotations`, full type hints on
  public functions.
- Async by default (`asyncio`); blocking calls go through `asyncio.to_thread`.
- Transport is ZeroMQ via `pyzmq` with `zmq.asyncio`. Socket patterns and
  high-water marks are decided per link in `docs/adr/0002-transport.md`; do
  not add a second transport.
- Messages are Pydantic v2 models serialized with msgpack. Image data travels
  as a separate raw ZMQ frame after the header, never inside the model.
- Every message carries a monotonic timestamp from the sensor service;
  latency is measured end to end from that stamp, never from wall-clock.
- No new runtime dependency without an ADR.
- Logging via `structlog`; never `print` in service code.
- Timestamps in nanoseconds as `int`; durations in seconds as `float`.

## Definition of done

A task is done only when all of these hold:

- `make check` passes locally; paste its last 10 lines in your summary.
- New behavior has a test; a bug fix has a regression test that fails
  without the fix.
- The reference app still runs end to end against the simulator.
- Any latency-sensitive change reports before/after p50 and p99 from
  `uv run pytest tests/integration -k latency`.

## Working rules

- Plan before editing anything in more than one service.
- `lmitechnologies/Gadget_Inspection_Template` is a design reference. You may
  read it to understand intent. Do not copy code from it unless
  `docs/adr/0001-origin.md` records its license as permitting reuse, and then
  only with attribution in the file header.
- Do not add files to the repo root. Config lives in `pyproject.toml`.
- Do not invent sensor models or PLC behavior; use the simulator in `sim/`
  and ask when the simulator lacks what you need.
- When a command fails, show the error and your diagnosis; do not retry
  with `--no-verify`, `|| true`, or by deleting the failing test.
