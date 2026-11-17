# AWS Local Dev

Run AWS Emulator and the StackPort web console on macOS with Docker Compose. The setup script configures a dedicated `local-emulator` AWS CLI profile.

## Prerequisites

- macOS with [Docker Desktop](https://docs.docker.com/desktop/setup/install/mac-install/) running. Docker Compose is included.
- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) and Git.
- Port `9297` available for AWS Emulator. StackPort uses `8081` by default.

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

| Option | What it does |
| --- | --- |
| **1. Permanent setup** | Copies the setup files to a directory you choose and starts both services. Press Enter to use `~/dev/aws-local-dev`. |
| **2. Transient setup** | Copies the setup files to a temporary directory and starts both services. |
| **3. Clean up** | Prompts before removing the shared stack, volume, local CLI profile, and certificate. |

If you leave the **Installation folder** prompt blank and press Enter, the script installs to `~/dev/aws-local-dev` (`$HOME/dev/aws-local-dev`). Running `./aws-emulator-setup --permanent` without a path uses the same default.

The first run builds the emulator image and pulls the StackPort image. The script opens StackPort once both services are ready and prints the setup path and restart command. Both installation modes use the same stack and profile. Emulated resources are held in memory and clear when the emulator stops.

### Run with flags

```bash
./aws-emulator-setup --permanent
./aws-emulator-setup --permanent "$HOME/projects/aws-local-dev"
./aws-emulator-setup --transient
```
<img width="437" height="154" alt="image" src="https://github.com/user-attachments/assets/6638fb95-6c7c-4932-ae38-6403f62f46da" />

## Open StackPort

Open [StackPort](http://127.0.0.1:8081) while the stack is running. On macOS, you can reopen the tab with `open http://127.0.0.1:8081`. Use the sidebar or search bar to find **S3** or **Lambda**.

The emulator listens at `https://127.0.0.1:9297`. If the StackPort port is busy, run `STACKPORT_UI_PORT=8082 ./aws-emulator-setup` and use the URL it prints.

## Try the AWS CLI

```bash
aws --profile local-emulator s3 mb s3://local-demo
aws --profile local-emulator s3 ls
```

Open **S3** in StackPort to see the bucket. To omit `--profile`, answer **yes** to the local shell prompt after setup. Type `exit` when you're done with that shell.

## Deploy a Lambda with Terraform

With the emulator running and [Terraform](https://developer.hashicorp.com/terraform/install) installed, create a separate directory for this example. Copy the following into `main.tf`.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
    archive = {
      source  = "hashicorp/archive"
      version = "~> 2.0"
    }
  }
}

variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "expected_account_id" {
  description = "AWS account ID Terraform is allowed to manage."
  type        = string
}

provider "aws" {
  region              = var.aws_region
  allowed_account_ids = [var.expected_account_id]
}

data "archive_file" "lambda" {
  type                    = "zip"
  output_path             = "${path.module}/hello-terraform.zip"
  source_content_filename = "lambda_function.py"

  source_content = <<-PYTHON
    def lambda_handler(event, context):
        name = event.get("name", "World")
        return {"message": f"Hello, {name}!"}
  PYTHON
}

resource "aws_iam_role" "lambda" {
  name = "hello-terraform-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_lambda_function" "hello" {
  function_name    = "hello-terraform"
  role             = aws_iam_role.lambda.arn
  runtime          = "python3.12"
  handler          = "lambda_function.lambda_handler"
  filename         = data.archive_file.lambda.output_path
  source_code_hash = data.archive_file.lambda.output_base64sha256
}

output "function_name" {
  value = aws_lambda_function.hello.function_name
}
```

In a terminal without cloud AWS credentials or endpoint overrides, run:

```bash
(
  export AWS_PROFILE=local-emulator
  export AWS_CA_BUNDLE="$HOME/.aws/local-emulator/ca.pem"
  export TF_VAR_expected_account_id=000000000000
  terraform init &&
    terraform plan -out=local.tfplan &&
    read -r -p "Apply this local plan? Type yes: " approval &&
    [ "$approval" = yes ] &&
    terraform apply local.tfplan
)
```

Review the plan for the Lambda and IAM role before applying. The profile supplies the emulator endpoint, and the account guard limits the deployment to the local account. Keep this Terraform state separate from any real AWS deployment.

### Invoke from StackPort

Open **Lambda → hello-terraform → Invoke**, enter `{"name":"Kod"}`, and click **Invoke**. The expected response is `{"message":"Hello, Kod!"}`.
<img width="1058" height="674" alt="image" src="https://github.com/user-attachments/assets/81167b8b-705f-4aed-ba2b-cb6cf7d599ec" />


### Invoke with curl

Lambda's Invoke API uses AWS Signature Version 4. With a version of curl that supports `--aws-sigv4`, sign the request using the generated credentials from the local profile.

```bash
(
  set -eu
  access_key="$(aws configure get aws_access_key_id --profile local-emulator)"
  secret_key="$(aws configure get aws_secret_access_key --profile local-emulator)"

  printf 'user = "%s:%s"\n' "$access_key" "$secret_key" |
    curl -q --config - --fail-with-body --silent --show-error \
      --aws-sigv4 'aws:amz:us-east-1:lambda' \
      --cacert "$HOME/.aws/local-emulator/ca.pem" \
      --header 'Content-Type: application/json' \
      --data '{"name":"Kod"}' \
      'https://127.0.0.1:9297/2015-03-31/functions/hello-terraform/invocations'
)
```

### Invoke from an application

For an application running directly on your Mac, configure its AWS SDK to use the `local-emulator` profile, region `us-east-1`, and endpoint `https://127.0.0.1:9297`. Configure the SDK to trust `~/.aws/local-emulator/ca.pem`, then call Lambda's `Invoke` with function name `hello-terraform` and a JSON payload such as `{"name":"Kod"}`. The SDK signs the request.

## Restart, update, and clean up

- **Restart** using the command printed by setup. For the default permanent installation, run `~/dev/aws-local-dev/aws-emulator-setup --permanent`. This refreshes the local profile and certificate.
- **Update** by running `git pull` in the clone, then running `./aws-emulator-setup` and choosing your existing permanent directory. The installer backs up changed setup files.
- **Clean up** by choosing option **3** or running `./aws-emulator-setup --cleanup` directly. This removes the shared stack, volume, local profile, and certificate. It clears emulated resources and leaves the setup files on disk. If you used custom AWS config or credentials paths, supply the same values during cleanup.

## Troubleshooting

| Issue | What to check |
| --- | --- |
| Docker is unavailable | Start Docker Desktop and run `docker info`. |
| The script returns `permission denied` | Run `chmod +x aws-emulator-setup` and retry. |
| StackPort won’t open | Use the printed URL. From the setup directory, check `docker compose ps` and `docker compose logs --tail=50`. |
| The local profile fails after a restart | Rerun setup to refresh the local credentials and certificate. |

## References

- [AWS Emulator Docker guide](https://docs.aws.amazon.com/emulator/latest/userguide/getting-started-docker.html)
- [StackPort guide for AWS Emulator](https://stackport.cloud/docs/guides/aws-emulator/)
- [Lambda Invoke API](https://docs.aws.amazon.com/lambda/latest/api/API_Invoke.html)
- [curl manual](https://curl.se/docs/manpage.html#--aws-sigv4)
