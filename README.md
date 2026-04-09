# Bazel Remote Cache (Flox Environment)

A Flox environment that runs [bazel-remote](https://github.com/buchgr/bazel-remote), providing an HTTP/gRPC remote cache for Bazel builds.

## Quick Start

```bash
# Start the cache server
flox activate --start-services

# Verify it's running
curl http://localhost:9090/status
```

The cache server will be available at:
- **HTTP**: `http://localhost:9090`
- **gRPC**: `grpc://localhost:9092`

## Configuration

Defaults can be overridden at activation time via environment variables:

| Variable | Default | Description |
|---|---|---|
| `BAZEL_REMOTE_HTTP_ADDRESS` | `0.0.0.0:9090` | HTTP listen address |
| `BAZEL_REMOTE_GRPC_ADDRESS` | `0.0.0.0:9092` | gRPC listen address |
| `BAZEL_REMOTE_MAX_SIZE` | `10` | Max disk cache size in GiB |

Example with overrides:

```bash
BAZEL_REMOTE_MAX_SIZE=50 BAZEL_REMOTE_HTTP_ADDRESS=127.0.0.1:8080 flox activate --start-services
```

Cache data is stored in `$FLOX_ENV_CACHE/bazel-remote/data` and persists across activations.

## Managing the Service

```bash
flox services status                  # Check status
flox services logs bazel-remote-cache # View logs
flox services restart bazel-remote-cache
flox services stop bazel-remote-cache
```

> **Note:** `flox services status` may show "Launching" even after the service is fully ready.
> To verify the cache server is running and accepting requests:
>
> ```bash
> curl http://localhost:9090/status
> ```
>
> A JSON response with `MaxSize`, `NumFiles`, etc. confirms the server is operational.

## Inspecting and Clearing the Cache

**Inspect** the cache size and file count:

```bash
# Server status (size, file count)
curl http://localhost:9090/status

# Disk usage
du -sh $FLOX_ENV_CACHE/bazel-remote/data
```

**Clear** the cache by stopping the service, removing the data, and restarting:

```bash
flox services stop bazel-remote-cache
rm -rf $FLOX_ENV_CACHE/bazel-remote/data
flox services start
```

The data directory is recreated automatically on service start.

## Configuring Bazel Clients

In any other environment running Bazel, add the following to the project's `.bazelrc` file (replace `CACHE_HOST` with the hostname or IP of the machine running this environment):

### HTTP remote cache

```
# .bazelrc
build --remote_cache=http://CACHE_HOST:9090
```

### gRPC remote cache (recommended, faster for large artifacts)

```
# .bazelrc
build --remote_cache=grpc://CACHE_HOST:9092
```

### Read-only mode (for CI or untrusted environments)

```
# .bazelrc
build --remote_cache=grpc://CACHE_HOST:9092
build --remote_upload_local_results=false
```

### Localhost (cache server and Bazel on the same machine)

```
# .bazelrc
build --remote_cache=grpc://localhost:9092
```

### Flox environment with Bazel configured to use the cache

In a separate Flox environment where you run Bazel builds, you can set the cache address via an environment variable and reference it in `.bazelrc`:

```toml
# manifest.toml of the build environment
[vars]
BAZEL_REMOTE_CACHE_URL = "grpc://localhost:9092"
```

```
# .bazelrc of the project being built
build --remote_cache=grpc://localhost:9092
```

Or pass it directly on the command line:

```bash
bazel build //... --remote_cache=grpc://localhost:9092
```

## Verifying the Cache is Working

After running a Bazel build with the remote cache configured:

1. Check that the cache has entries:
   ```bash
   curl http://CACHE_HOST:9090/status
   ```
   `NumFiles` should be greater than 0.

2. Clean and rebuild — the second build should be significantly faster:
   ```bash
   bazel clean
   bazel build //...
   ```
   You should see cache hits in the build output.
