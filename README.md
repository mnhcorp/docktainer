# Docktainer: A Docker-like CLI Wrapper for Apptainer

**Docktainer** provides a familiar Docker-like command-line interface (CLI) for interacting with Apptainer (formerly Singularity). It's designed to help users, especially those in HPC environments, leverage Apptainer's strengths while using a more Docker-centric command syntax.

This tool translates common Docker commands into their Apptainer equivalents, managing local SIF image storage and basic state for detached Apptainer instances.

## Why Docktainer?

- **Familiar UX:** If you're used to Docker commands, `docktainer` offers a smoother transition to using Apptainer.
- **Simplified Apptainer Usage:** Abstracts some of Apptainer's specific command structures.
- **HPC Friendly:** Apptainer is often preferred in HPC environments for security and integration. Docktainer makes it easier to use in these contexts if you're coming from a Docker background.
- **Local SIF Management:** Automatically stores pulled and built SIF images in `~/.docktainer/images/`.
- **Instance Management:** Basic support for starting, listing, exec-ing into, and stopping detached services as Apptainer instances.

## Prerequisites

- **Linux Environment:** Docktainer is designed for Linux.
- **Apptainer Installed:** You must have a working Apptainer installation. Verify with `apptainer --version`.
- **Python 3.6+:** The `docktainer.py` script requires Python 3.

## Installation

1. **Download/Save the Script:**  
    Obtain the `docktainer.py` script and save it to your system.

2. **Make it Executable:**
    ```bash
    chmod +x docktainer
    ```

3. **Place it in your PATH (Recommended):**  
    To run `docktainer` from any directory, move it to a location in your system's `PATH` (e.g., `/usr/local/bin`) and optionally rename it.
    ```bash
    sudo mv docktainer /usr/local/bin/docktainer
    ```
    Alternatively, create a symbolic link or add its directory to your `PATH` environment variable. After this, you can invoke it simply as `docktainer`.

## Basic Usage

The general command structure is:

```bash
docktainer [GLOBAL_OPTIONS] <COMMAND> [COMMAND_OPTIONS] [ARGUMENTS...]
```

**Global Options:**

- `-V`, `--verbose`: Enable verbose output. This is highly recommended when first using the tool or for debugging, as it shows the underlying Apptainer commands being generated and executed.

**Key Commands:**

- `pull`: Pull an image from Docker Hub (converted to SIF).
- `images`: List locally stored SIF images.
- `run`: Run a command in a container or start/manage an Apptainer instance.
- `ps`: List running Apptainer instances managed by Docktainer.
- `exec`: Execute a command in a running detached Apptainer instance.
- `stop`: Stop a running detached Apptainer instance.
- `build`: Build a SIF image from a (simplified) Dockerfile.
- `rmi`: Remove a local SIF image.

Refer to `docktainer --help` for a full list of commands and options.

## How it Works (Briefly)

Docktainer works by:

- Parsing Docker-like commands and options.
- Translating these into equivalent Apptainer commands and options.
- For `pull` and `build`, it manages SIF images in `~/.docktainer/images/`.
- For `run -d` (detached mode), it uses `apptainer instance start` to create named Apptainer instances and tracks their state in `~/.docktainer/instances.json`.
- Commands like `ps`, `exec`, and `stop` then operate on these managed instances or PIDs.

## Examples

### 1. Pulling Images

```bash
# Pull the latest Alpine image
docktainer pull alpine:latest

# Pull a specific Ubuntu version
docktainer pull ubuntu:22.04

# Force pull, overwriting local SIF if it exists
docktainer pull --force alpine:latest
```

Images are stored as `.sif` files in `~/.docktainer/images/`.

### 2. Listing Local Images

```bash
docktainer images
```

Output will resemble:

```
REPOSITORY          TAG                 IMAGE ID         CREATED                   SIZE
ubuntu              22.04               abcdef123456     2025-05-16 10:00:00       123.45MB
alpine              latest              fedcba654321     2025-05-16 09:50:00       5.20MB
```

### 3. Running Containers

#### a) Executing a one-off command

```bash
docktainer run alpine:latest cat /etc/os-release
docktainer run ubuntu:22.04 echo "Hello from Ubuntu"
```

#### b) Interactive shell

```bash
docktainer run -it alpine:latest /bin/sh
# Inside the shell:
#   whoami
#   ls -la /
#   exit
```

#### c) Volume mounting

```bash
mkdir -p ./mydata
echo "host content" > ./mydata/test.txt
docktainer run -it -v "$(pwd)/mydata:/shared" alpine:latest /bin/sh
# Inside the shell:
#   cat /shared/test.txt
#   echo "container content" > /shared/container_made.txt
#   exit
# Back on host:
#   cat ./mydata/container_made.txt
```

#### d) Detached Mode (as Apptainer Instance)

This starts the container's default runscript/startscript as a background Apptainer instance.

```bash
# Example: Use an image that runs a service or a long-running task.
# For a simple test, ensure your SIF's %runscript or %startscript does something persistent.
# E.g., a SIF built with `FROM alpine %runscript exec sleep infinity` named `looper.sif`
# docktainer run -d --name my-looper ./looper.sif

# Using a pre-built image like nginx (its default command will run)
docktainer run -d --name webserver nginx:alpine 

# The command 'sleep 300' below is for display purposes in `docktainer ps` for this example.
# The actual running process inside the instance is defined by alpine:latest's default entrypoint/cmd.
docktainer run -d --name sleeper alpine:latest sleep 300 
```

Output will be the instance name (e.g., `webserver` or `sleeper`).

### 4. Listing Running Instances (`ps`)

```bash
docktainer ps
```

Shows instances started with `docktainer run -d` and foreground processes started by the current docktainer session (while they are running).

```
ID/NAME          IMAGE                          COMMAND (at start)    CREATED        STATUS   USER      PID
my-looper        looper.sif                     <runscript>           10s ago        Up       youruser  12345
sleeper          alpine_latest.sif              sleep 300             2m ago         Up       youruser  12348
```

### 5. Executing Commands in a Running Detached Instance (`exec`)

This targets instances started with `docktainer run -d --name ...`.

```bash
# Assuming 'my-looper' instance is running
docktainer exec -it my-looper /bin/sh 
# Inside: ps aux
# exit

docktainer exec my-looper date
```

### 6. Stopping a Detached Instance (`stop`)

```bash
docktainer stop my-looper
docktainer ps 
```

The `my-looper` instance should no longer be listed by `docktainer ps` (or will be cleaned up from the state file on the next `ps` run if Apptainer confirms it's stopped).

### 7. Building Images from a Dockerfile (`build`)

Docktainer includes a simplified Dockerfile parser. It supports common instructions like `FROM`, `RUN`, `ENV`, `COPY` (basic cases), `WORKDIR`, `LABEL`, `CMD`, and `ENTRYPOINT`.

Create a directory, e.g., `my_docker_app/`.

Inside `my_docker_app/`, create a `Dockerfile`:

```dockerfile
FROM alpine:3.18
RUN apk add --no-cache curl
ENV GREETING="Hello from Docktainer Build"
COPY data.txt /app/data.txt
WORKDIR /app
CMD ["sh", "-c", "echo $GREETING && echo --- && cat /app/data.txt && curl -s ifconfig.me/host"]
```

Create `my_docker_app/data.txt`:

```
This is some data.
```

Build the image (from outside the `my_docker_app` directory):

```bash
docktainer build -t myapp:1.0 ./my_docker_app
```

A `myapp_1.0.sif` (or similar, based on tag) will be created in `~/.docktainer/images/`.

Run your built image:

```bash
docktainer run myapp:1.0
```

### 8. GPU Usage (`run --gpus`)

Requires NVIDIA drivers and Apptainer configured for NVIDIA GPU support on the host.

```bash
# Pull a CUDA-enabled image (use a specific tag, NOT 'latest' for nvidia/cuda)
docktainer pull nvidia/cuda:12.2.0-base-ubuntu22.04

# Run nvidia-smi using all available GPUs
docktainer run --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

# Run nvidia-smi using only GPU 0 (replace '0' with your GPU index)
docktainer run --gpus 0 nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

# Run a detached instance with GPU access to specific GPUs
docktainer run -d --name cuda_job --gpus 0,1 nvidia/cuda:12.2.0-base-ubuntu22.04
# (The instance will run the default command of the nvidia/cuda image)
```

This passes `--nv` to Apptainer and sets `NVIDIA_VISIBLE_DEVICES` if specific GPUs are requested.

### 9. Removing Images (`rmi`)

```bash
docktainer rmi myapp:1.0
docktainer rmi alpine:latest
```

## State Files

- **SIF Images:** `~/.docktainer/images/`
- **Managed Instance State:** `~/.docktainer/instances.json` (tracks instances started with `run -d` and foreground processes for `ps` cleanup).

## Troubleshooting

- **`docktainer ps` shows unexpected results or errors:**
  - Ensure `apptainer instance list --json` works correctly on your system.
  - Use `docktainer -V ps` to see verbose logs.
  - The state file `~/.docktainer/instances.json` might sometimes need manual inspection or clearing (`rm ~/.docktainer/instances.json`) if it gets into an inconsistent state, especially during development or after system crashes.

- **`nvidia/cuda:latest` pull fails:**  
  NVIDIA does not provide a generic `latest` tag for `nvidia/cuda` on Docker Hub. Always use a specific version tag (e.g., `12.2.0-base-ubuntu22.04`).

- **Command not found:**  
  Ensure `apptainer` and `docktainer` are in your system's `PATH`.

## Limitations

- **Simplified Build:** The Dockerfile parser is basic and may not support all Dockerfile instructions or complex syntaxes (e.g., multi-stage builds, advanced `ARG` usage).
- **Not a Full Docker Replacement:** Docktainer translates a subset of Docker commands. It does not replicate Docker's daemon, complex networking model (Apptainer typically uses host networking by default), or advanced volume management beyond simple bind mounts.
- **Apptainer's Behavior:** Ultimately, commands are executed by Apptainer, so its behavior, security model, and specific version capabilities apply.
- **Instance Command for `run -d`:** When using `docktainer run -d <image> [command]`, the `[command]` is primarily for display in `docktainer ps`. The actual process run by the Apptainer instance is determined by the SIF image's `%runscript` or `%startscript`.

## Contributing / Issues

This is a utility script.
