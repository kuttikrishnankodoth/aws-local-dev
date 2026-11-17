# AWS local development

Install [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) and start Docker Desktop on your Mac. From the project directory, run:

```sh
./aws-emulator-setup
```

The script asks what you would like to do. Choose **Clean up** to remove the existing stack and CLI profile, or choose **Permanent** to install into `~/dev/aws-local-dev`, or **Transient** to use a new folder under your system temporary directory. You can also pass `--permanent` or `--transient` directly. For daily use, run `~/dev/aws-local-dev/aws-emulator-setup` to start the permanent installation again.

After startup, the script prints the full paths to the setup files and AWS CLI profile files, plus a command to run it again.

Both choices use the same Docker Compose project and the same local ports. The choice controls where the setup files live. Emulated AWS resources are stored in memory and are cleared when the emulator stops.

The first run builds AWS Emulator and pulls StackPort. The script waits for both services and opens StackPort in your browser. Start at http://127.0.0.1:8081. The local AWS API is at https://127.0.0.1:9297. StackPort can use its API Explorer to create and inspect resources. The setup does not provision project resources. Docker Desktop includes Docker Compose. Git is needed if cloning the repo.

The script creates or refreshes the dedicated `local-emulator` CLI profile with the emulator endpoint, region, TLS certificate, and generated credentials. Rerun the script after an emulator restart to refresh the profile. Use a shell without AWS endpoint or TLS environment overrides when working with this profile.

```sh
aws --profile local-emulator s3 mb s3://local-demo
aws --profile local-emulator s3 ls
```

If the UI port is busy, use `STACKPORT_UI_PORT=8082 ./aws-emulator-setup`. Port 9297 must be available for the emulator. Logs are available with `docker compose logs -f`.

Run `docker compose down` to stop both services. Emulated resources are stored in memory and are cleared when the emulator stops.

## Choose a permanent installation folder

Select **Permanent setup** and enter the folder where the setup files should live. Press Enter to use `~/dev/aws-local-dev`. You can also specify a folder directly:

```sh
./aws-emulator-setup --permanent "$HOME/projects/aws-local-dev"
```

Use the restart command printed at the end to reopen that installation.

## Destroy a permanent installation

Run the cleanup flag to stop both containers, remove their shared volume, and delete the `local-emulator` CLI profile and TLS certificate. This clears emulated resources such as S3 buckets. If you used custom AWS config or credentials paths, supply the same environment variables for cleanup.

```sh
~/dev/aws-local-dev/aws-emulator-setup --cleanup
```

For a custom permanent folder, replace `~/dev/aws-local-dev` with the setup folder printed at startup in the cleanup and folder removal commands.

For a transient installation, run `./aws-emulator-setup --cleanup` from the extracted package or the setup folder printed at startup.

To also remove the installed setup files, leave the folder first, then delete it. This deletes any files you added to that folder.

```sh
cd "$HOME"
rm -r "$HOME/dev/aws-local-dev"
```

The emulator build uses AWS's latest LTS package, and Compose uses StackPort's latest image. Fresh installs can pick up newer releases.

- [AWS Emulator Docker guide](https://docs.aws.amazon.com/emulator/latest/userguide/getting-started-docker.html)
- [StackPort AWS Emulator guide](https://stackport.cloud/docs/guides/aws-emulator/)

After startup, answer **yes** to the AWS profile prompt to open a shell
with `AWS_PROFILE=local-emulator`. Run commands such as `aws s3 ls`
without a profile flag. Type `exit` to return to your original shell.

To update a permanent installation, run the newer package with `--permanent`. Changed setup files are backed up in a `backup.*` folder inside the installation.
# aws-local-dev
