# Docksmith

Docksmith is a lightweight, Docker-inspired container image builder and runtime implemented in Go. It parses a `Docksmithfile`, builds content-addressed filesystem layers, stores image manifests locally, reuses cached build steps, and runs commands from assembled image filesystems.

> **Project status:** Docksmith is an educational/local containerization project. It currently uses host OS processes and temporary directories for isolation rather than Linux namespaces, cgroups, or a registry-backed image store.

## Features

- Docker-inspired `Docksmithfile` instructions:
  - `FROM`
  - `COPY`
  - `RUN`
  - `WORKDIR`
  - `ENV`
  - `CMD`
- Content-addressed layers using SHA-256 digests.
- Incremental filesystem layers containing only changed or newly created files.
- Build cache persisted under `~/.docksmith/cache/cache.json`.
- Local image manifests and layers stored under `~/.docksmith/`.
- Cache-hit and cache-miss reporting during builds.
- Container execution with working-directory, command, and environment-variable support.
- Local image listing and removal through a CLI.

## Tech Stack

- Go 1.22.2
- Go standard library packages including `archive/tar`, `crypto/sha256`, `os/exec`, `filepath`, and `encoding/json`
- Python sample application for validating image builds and runtime environment variables

## Repository Structure

```text
.
├── main.go                 # CLI entry point and command-line argument parsing
├── Docksmithfile           # Example image definition
├── builder/                # Build pipeline, manifests, cache keys, and image configuration
├── parser/                 # Docksmithfile parser and instruction validation
├── layers/                 # Filesystem snapshots, delta layers, tar archives, and digests
├── cache/                  # Persistent JSON-backed build-cache index
├── runtime/                # Command execution and filesystem-delta capture
├── cli/                    # Image build, run, list, remove, and directory setup commands
├── sample_app/             # Example Python application and Docksmithfile
└── go.mod                  # Go module definition
```

## How It Works

1. The CLI parses a build command and initializes `~/.docksmith/images`, `~/.docksmith/layers`, and `~/.docksmith/cache`.
2. `parser.ParseFile` converts each `Docksmithfile` instruction into a typed instruction.
3. `builder.Build` processes the instructions, inherits local base-image metadata, and tracks environment variables, working directory, command, and layers.
4. `COPY` and `RUN` steps execute against an assembled temporary filesystem. Before-and-after snapshots are compared to create a delta tar layer.
5. Each layer is hashed with SHA-256 and stored by digest. Deterministic cache keys combine the previous layer, instruction, environment, working directory, and `COPY` source contents.
6. The final image manifest is stored locally. `docksmith run` extracts the ordered layers into a temporary directory and runs the configured command with the requested environment.

## Getting Started

### Prerequisites

- Go 1.22.2 or later
- A Unix-like environment with `/bin/sh` for `RUN` instructions
- Commands referenced by `RUN` and `CMD` available on the host system

### Build the CLI

```bash
go build -o docksmith .
```

### Build an image

The repository includes an example `Docksmithfile`. Because `FROM` resolves images from Docksmith's local image store, a base image manifest must already exist locally before using `FROM alpine`.

```bash
./docksmith build -t sample:latest .
./docksmith images
```

Disable the build cache when needed:

```bash
./docksmith build -t sample:latest --no-cache .
```

### Run an image

```bash
./docksmith run sample:latest
./docksmith run -e NAME=Bhavana sample:latest
./docksmith run sample:latest python3 app.py
```

### Remove an image

```bash
./docksmith rmi sample:latest
```

## Docksmithfile Example

```dockerfile
FROM alpine
WORKDIR /app
COPY . /app
RUN echo "Build complete"
ENV NAME=Docksmith
CMD ["sh", "-c", "echo Hello, $NAME"]
```

## Local Storage

```text
~/.docksmith/
├── images/    # JSON image manifests
├── layers/    # SHA-256-addressed tar layers
└── cache/     # Persistent build cache index
```

## Limitations and Future Improvements

- No image registry, pull/push workflow, or automatic base-image download.
- Runtime isolation is process/filesystem based; Linux namespaces, cgroups, capabilities, and networking are not implemented.
- Deleted files are not represented as whiteout layers.
- The supported instruction set is intentionally smaller than Dockerfile syntax.
- Automated tests and CI workflows should be added as the project matures.

## License

No license has been specified yet.
