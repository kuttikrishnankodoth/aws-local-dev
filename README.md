# AWS Local Dev

Run AWS Emulator and the StackPort web console on macOS with Docker Compose. The setup script starts both services and configures a dedicated `local-emulator` AWS CLI profile so you can create and inspect local resources without an AWS account.

## Prerequisites

- **macOS** with [Docker Desktop](https://docs.docker.com/desktop/setup/install/mac-install/) installed and running. Docker Compose is included.
- **[AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)** installed.
- **Git** installed to clone the repository.
- **Port `9297`** available for AWS Emulator. StackPort uses port `8081` by default.

Verify the prerequisites in Terminal:

```bash
git --version
docker info --format '{{.ServerVersion}}'
docker compose version
aws --version
```

## Clone and set up

```bash
git clone https://github.com/kuttikrishnankodoth/aws-local-dev.git
cd aws-local-dev
./aws-emulator-setup
```

Choose an option from the menu:

| Option | What it does |
| --- | --- |
| **1. Permanent setup** | Installs the setup files in a folder you choose and starts both services. Press Enter to use `~/dev/aws-local-dev`. |
| **2. Transient setup** | Copies the setup files into a temporary folder and starts both services. Use this for a quick test. |
| **3. Clean up** | Asks for confirmation before removing the Docker stack, shared volume, CLI profile, and TLS certificate. |

The first run builds the emulator image and pulls the StackPort image. Once both services are ready, the script opens StackPort and prints the setup directory and a restart command.

Permanent and transient setups share the same Docker Compose project and AWS CLI profile. The choice controls where the setup files are stored. The emulator holds resources such as buckets and functions in memory, so they are cleared when the emulator stops.

### Run with flags

Use these commands to skip the setup menu:

```bash
# Permanent setup in the default directory
./aws-emulator-setup --permanent

# Permanent setup in a custom directory
./aws-emulator-setup --permanent "$HOME/projects/aws-local-dev"

# Transient setup
./aws-emulator-setup --transient

# Remove the local stack and profile directly
./aws-emulator-setup --cleanup
```

## Open StackPort

Open [StackPort](http://127.0.0.1:8081) in your browser while the stack is running. If you close the tab, reopen it with:

```bash
open http://127.0.0.1:8081
```

Use the sidebar or search bar to find a service. Open **S3** to browse local buckets, or **Lambda** to inspect and invoke a function you've deployed.

AWS Emulator listens at `https://127.0.0.1:9297`. If port `8081` is busy, choose a different StackPort port and use the URL printed by the script:

```bash
STACKPORT_UI_PORT=8082 ./aws-emulator-setup
```

## Try the AWS CLI

Create an S3 bucket and list your local buckets:

```bash
aws --profile local-emulator s3 mb s3://local-demo
aws --profile local-emulator s3 ls
```

Open **S3** in StackPort and refresh the bucket list to see `local-demo`.

### Run commands without `--profile`

When setup asks whether to use the `local-emulator` profile, answer **yes**. The script opens your usual shell with the local profile selected. You can then run:

```bash
aws s3 ls
```

Type `exit` to return to your original shell.

## Restart, update, and clean up

### Restart

For the default permanent installation, run:

```bash
~/dev/aws-local-dev/aws-emulator-setup --permanent
```

For a custom or transient installation, use the restart command printed by the setup script. Rerunning setup refreshes the CLI profile with the emulator’s credentials and TLS certificate.

### Update

From the cloned repository, run:

```bash
git pull
./aws-emulator-setup
```

Choose **Permanent setup** and enter your existing installation directory. The installer backs up changed setup files before replacing them.

### Clean up

Run `./aws-emulator-setup` and choose **3. Clean up**. Cleanup stops both services, removes their shared Docker volume, and deletes the `local-emulator` CLI profile and TLS certificate. This clears local emulated resources. The setup files remain available for reuse.

If you used custom `AWS_CONFIG_FILE` or `AWS_SHARED_CREDENTIALS_FILE` paths during setup, use the same values during cleanup.

## Troubleshooting

| Issue | What to check |
| --- | --- |
| Docker is unavailable | Start Docker Desktop and run `docker info` to verify it’s ready. |
| `aws` command is missing | Install AWS CLI v2 and verify with `aws --version`. |
| The script returns `permission denied` | Run `chmod +x aws-emulator-setup` from the repository directory and try again. |
| StackPort won’t open | Check the URL printed by the script. From the setup directory, run `docker compose ps` and `docker compose logs --tail=50`. |
| The local profile fails after a restart | Rerun setup to refresh the emulator credentials and TLS certificate. |

## Upstream projects

- [AWS Emulator Docker guide](https://docs.aws.amazon.com/emulator/latest/userguide/getting-started-docker.html)
- [StackPort guide for AWS Emulator](https://stackport.cloud/docs/guides/aws-emulator/)
