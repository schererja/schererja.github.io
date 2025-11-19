---
layout: post
title: "SMIDR: Rethinking Embedded Linux Builds with Containers and gRPC"
date: "2024-10-24 7:25:00 -0400"
categories: ["development", "life", "education"]
tags: ["development",  "life", "education", "electronics"]
---
# SMIDR: Rethinking Embedded Linux Builds with Containers and gRPC

Every embedded developer has been there: you need to build three different variants of your Yocto image—one for dev, one for QA, and one for production. That's 360GB of disk space, three terminals, and a full day of waiting. And heaven forbid you need to debug a build failure—you're digging through cryptic BitBake errors with no idea what went wrong.

I got tired of this workflow, so I built [SMIDR](https://github.com/schererja/smidr)—a tool that treats Yocto builds like Docker treats application deployments: containerized, reproducible, and manageable.

## The Pain Points

If you've worked with Yocto/BitBake for any length of time, these problems sound familiar:

**Storage Bloat**: Each build environment needs 120GB+. Building multiple image variants? Multiply that by N. A typical project can easily consume 500GB+ of disk space with redundant downloads and build artifacts.

**Build State Corruption**: Change one layer, and suddenly BitBake throws mysterious errors. The "solution"? Wipe everything and rebuild from scratch—there goes 4 hours.

**Terminal Blocking**: Start a build and your terminal is locked for hours. Want to check on it remotely? Hope you remembered to start it in `tmux`.

**No Parallelization**: Need to build for multiple targets? Either open multiple terminals and pray your machine doesn't run out of RAM, or build them sequentially.

**Cryptic Errors**: Parser failures, dependency resolution errors, and task failures with no clear indication of what actually went wrong.

## The SMIDR Philosophy

SMIDR is built on three core principles:

### 1. Container-Native from Day One

Every build runs in an isolated Docker container with exactly the dependencies it needs. No more "it works on my machine"—if it builds once, it builds everywhere.

The container setup is smart:

- Automatically mounts shared download and sstate-cache directories
- Handles layer discovery and BBLAYERS configuration
- Sets proper TMPDIR and DEPLOY paths
- Manages user permissions so artifacts are accessible outside the container

### 2. Intelligent Caching

Instead of duplicating 120GB per build, SMIDR shares common resources:

```yaml
directories:
  layers: ~/.smidr/layers           # Shared layer repos
  downloads: ~/.smidr/downloads      # Shared DL_DIR (source archives)
  sstate: ~/.smidr/sstate-cache     # Shared build state
```

With this setup:

- **90% space savings**: Three 120GB builds become one 40GB cache + small per-build deltas
- **Faster builds**: Shared sstate means rebuilds only compile what actually changed
- **Better CI**: Cache once, build many times

### 3. Developer Experience First

The CLI is designed around common workflows:

```bash
# Initialize a new project
smidr init my-project

# Build with clear progress
smidr build --target core-image-minimal

# Check what happened
smidr status
smidr logs

# Manage artifacts
smidr artifacts list
smidr artifacts copy <build-id> ./output
```

Configuration is YAML-based and readable:

```yaml
name: toradex-custom-image
description: "Custom Toradex image with our stack"

base:
  provider: toradex
  machine: verdin-imx8mp
  distro: tdx-xwayland
  version: "6.0.0"

layers:
  - name: meta-toradex-bsp-common
    git: https://git.toradex.com/meta-toradex-bsp-common
    branch: kirkstone-6.x.y

  - name: meta-mycompany
    path: ./layers/meta-mycompany

build:
  image: core-image-weston
  bb_number_threads: 8
  parallel_make: 8
  extra_packages:
    - python3
    - docker
```

No more hunting through 12 different `conf` files to change parallelism settings or add a package.

## From CLI to Daemon: The Architecture Evolution

SMIDR started as a simple CLI tool, but as I used it for real projects, I hit a wall: **Yocto builds take hours, and blocking a terminal isn't scalable**.

That's when I decided to split it into a client-server architecture with gRPC.

### The Problem with the CLI-Only Approach

The original `smidr build` command did everything:

1. Parse config
2. Fetch layers
3. Set up container
4. Run BitBake
5. Extract artifacts
6. Stream logs to stdout

This worked fine for single builds, but:

- You couldn't start a build and close your laptop
- No way to monitor builds remotely
- Couldn't easily parallelize multiple builds
- No build history or status tracking

### Enter the Daemon

I introduced a gRPC server that runs as a persistent daemon:

```bash
# Start the daemon (runs in background)
smidr daemon --address :50051

# Submit a build (returns immediately)
smidr client start --config prod.yaml --target core-image-minimal
# Output: Build ID: acme-a3f9c2b1

# Check status anytime
smidr client status --build-id acme-a3f9c2b1

# Stream logs (live or historical)
smidr client logs --build-id acme-a3f9c2b1 --follow

# List all builds
smidr client list

# Cancel if needed
smidr client cancel --build-id acme-a3f9c2b1
```

The gRPC API is clean and purpose-built:

```protobuf
service Smidr {
  rpc StartBuild(StartBuildRequest) returns (StartBuildResponse);
  rpc GetBuildStatus(BuildStatusRequest) returns (BuildStatusResponse);
  rpc StreamLogs(StreamLogsRequest) returns (stream LogLine);
  rpc ListBuilds(ListBuildsRequest) returns (ListBuildsResponse);
  rpc CancelBuild(CancelBuildRequest) returns (CancelBuildResponse);
  rpc ListArtifacts(ListArtifactsRequest) returns (ListArtifactsResponse);
}
```

### The Refactoring Challenge: Avoiding Duplication

With both a CLI and daemon, I faced a choice:

1. Copy the build logic into both (maintainability nightmare)
2. Extract it into a shared component

I chose option 2 and created `internal/build/runner.go`—a shared build orchestrator used by both paths:

```go
type Runner struct {
    dockerManager *docker.DockerManager
}

type BuildOptions struct {
    Target      string
    Customer    string
    ForceClean  bool
    ForceImage  bool
}

func (r *Runner) Run(
    ctx context.Context,
    cfg *config.Config,
    opts BuildOptions,
    logSink LogSink,
) (*BuildResult, error)
```

The `LogSink` interface was the key to decoupling:

```go
type LogSink interface {
    Write(stream, line string)
}
```

Now:

- **CLI**: LogSink writes to stdout and a file
- **Daemon**: LogSink broadcasts to gRPC streaming subscribers

This dropped the CLI from ~900 lines to ~300, and the daemon shares the exact same build logic.

### How the Daemon Works

The daemon manages builds in memory:

```go
type BuildInfo struct {
    ID          string
    State       v1.BuildState
    Config      string
    Target      string
    StartTime   time.Time
    ExitCode    int32
    Error       string
    LogBuffer   []*v1.LogLine
    Subscribers map[string]chan *v1.LogLine
    CancelFunc  context.CancelFunc
}
```

When a client calls `StartBuild`:

1. Generate a unique build ID: `{customer}-{8-char-uuid}`
2. Create a cancellable context
3. Spawn a goroutine to execute the build
4. Return immediately to the client

The build goroutine:

1. Updates state: `QUEUED → PREPARING → BUILDING`
2. Calls `runner.Run()` with a log sink that broadcasts to subscribers
3. Updates state: `COMPLETED` or `FAILED`
4. Stores the result

Log streaming was surprisingly elegant. When a client subscribes:

```go
func (s *Server) StreamLogs(
    req *v1.StreamLogsRequest,
    stream v1.Smidr_StreamLogsServer,
) error {
    // Send buffered logs first (for builds already running)
    for _, line := range build.LogBuffer {
        if err := stream.Send(line); err != nil {
            return err
        }
    }

    // Subscribe to new logs if build is active
    if build.State == v1.BuildState_BUILDING {
        logChan := make(chan *v1.LogLine, 100)
        build.Subscribers[subID] = logChan
        defer delete(build.Subscribers, subID)

        for line := range logChan {
            if err := stream.Send(line); err != nil {
                return err
            }
        }
    }
}
```

This means:

- Clients can attach to live builds and see logs in real-time
- Historical builds show buffered logs instantly
- Multiple clients can stream the same build's logs

## Real-World Impact

Here's what this architecture enables:

**CI/CD Integration**:

```bash
# In GitLab CI / GitHub Actions
smidr client start --config ci.yaml --target production-image --customer ${CI_PROJECT_NAME}
BUILD_ID=$(smidr client list --limit 1 | grep "Build ID" | awk '{print $3}')
smidr client logs --build-id $BUILD_ID --follow
```

**Multi-Variant Builds**:

```bash
# Start three builds in parallel
smidr client start --config dev.yaml --target dev-image --customer dev &
smidr client start --config qa.yaml --target qa-image --customer qa &
smidr client start --config prod.yaml --target prod-image --customer prod &

# Monitor all of them
smidr client list
```

**Remote Monitoring**:

```bash
# Start build on your workstation
smidr client start --config config.yaml --target my-image

# Close laptop, go get coffee

# Check status from phone using SSH
ssh build-server "smidr client status --build-id <id>"
```

## Architecture Deep Dive

SMIDR is written in Go and structured around clean separation of concerns:

```
internal/
├── build/              # Shared build runner
│   └── runner.go       # Core orchestration logic
├── cli/                # CLI commands
│   ├── build/          # Build command
│   └── client/         # Daemon client commands
├── daemon/             # gRPC server
│   └── server.go       # Build management and streaming
├── container/          # Docker abstraction
│   └── docker/         # DockerManager implementation
├── bitbake/            # BitBake execution
│   └── executor.go     # BitBake runner with streaming
├── source/             # Layer fetching
│   └── fetcher.go      # Git clone/update with caching
├── artifacts/          # Artifact management
│   └── manager.go      # Extraction and organization
└── config/             # Configuration
    └── config.go       # YAML parsing and validation
```

Key components:

**DockerManager** handles all container operations:

- Image pulling and validation
- Container creation with proper mounts
- Execution with streaming output
- Resource limits (CPU/memory)

**BitBakeExecutor** manages the build process:

- Generates `local.conf` and `bblayers.conf`
- Runs prefetch and build tasks
- Handles errors with retry logic (e.g., `cleansstate` on recipe failures)
- Streams logs in real-time

**SourceFetcher** handles layer management:

- Clones repositories with lock files to avoid races
- Updates existing clones intelligently
- Supports eviction and cleanup
- Auto-discovers sublayers (e.g., `meta-openembedded/meta-oe`)

## Lessons Learned

Building SMIDR taught me several important lessons:

### 1. Extract Shared Logic Early

Don't wait until you have duplication. The moment I saw the daemon would need build orchestration, I created the runner. This saved weeks of maintenance.

### 2. Design Interfaces for Decoupling

The `LogSink` interface let me reuse the entire build pipeline without coupling to output mechanisms. This pattern appears throughout SMIDR—small interfaces enable big flexibility.

### 3. gRPC Streaming is a Superpower

Server-side streaming with `stream v1.Smidr_StreamLogsServer` made real-time log streaming trivial. It handles backpressure, client disconnects, and reconnects naturally.

### 4. Context Propagation is Critical

Passing `context.Context` through the entire pipeline means cancellation (Ctrl+C in CLI, or `CancelBuild` RPC) just works. The build goroutine, container execution, and BitBake all respect cancellation.

### 5. Organize by Feature, Not File Type

I initially had all CLI commands in `internal/cli/*.go`. As it grew to 15+ files, I reorganized:

```
internal/cli/
├── build/              # All build-related code
├── client/             # All daemon client commands
├── daemon.go
├── init.go
└── artifacts.go
```

This makes it easy to find related code and understand boundaries.

### 6. Cache Everything, Trust Nothing

Yocto builds are expensive. SMIDR aggressively caches:

- Downloaded source tarballs (DL_DIR)
- Compiled build state (sstate-cache)
- Cloned layer repositories

But it also validates everything:

- Checksums on downloads
- Git refs on layers
- Container image existence

## What's Next

The daemon foundation is solid, but there's more to build:

**Short term:**

- Complete artifact download API
- Build queue management for concurrent builds
- Persistent storage (save build history to disk/database)

**Medium term:**

- Authentication (mTLS or token-based)
- Web UI (React + gRPC-Web)
- Build scheduling and webhooks
- Advanced caching strategies (artifact proxies, CDN integration)

**Long term:**

- Distributed builds across multiple machines
- Cloud build service (AWS/GCP integration)
- Multi-vendor BSP support (NXP, TI, Qualcomm, etc.)
- Integration with CI/CD platforms as first-class plugins

## Try It Yourself

SMIDR is open source: [github.com/schererja/smidr](https://github.com/schererja/smidr)

Get started:

```bash
# Clone and build
git clone https://github.com/schererja/smidr.git
cd smidr
make build

# Try the CLI
./smidr init my-project
cd my-project
# Edit smidr.yaml for your target
./smidr build

# Or try the daemon
./smidr daemon --address :50051

# In another terminal
./smidr client start --config config.yaml --target core-image-minimal
./smidr client list
./smidr client logs --build-id <id> --follow
```

## The Philosophy

Building SMIDR taught me that **good tools solve real problems without adding ceremony**.

Yocto is powerful but complex. SMIDR doesn't hide that complexity—it organizes it. Instead of fighting with build environments and cryptic errors, you spend time on what matters: building great products.

The daemon pattern opened up possibilities I hadn't considered when I started with a simple CLI. Remote builds, parallel execution, and real-time monitoring all emerged naturally once the architecture supported them.

If you're working on embedded Linux, give SMIDR a try. And if you're building any kind of long-running build system, consider the daemon pattern—your future self will thank you.

---

*Questions? Thoughts? Open an issue on [GitHub](https://github.com/schererja/smidr/issues) or reach out on [LinkedIn](https://linkedin.com/in/jasonscherer).*
