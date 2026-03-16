# Jenkins AMI with Packer

Builds an AWS AMI that comes preconfigured with Jenkins and tooling used for CI/CD workflows.

## What this repo does

This project uses **Packer** to create a reusable Jenkins AMI on Ubuntu. During provisioning it:

- Installs Jenkins and disables the setup wizard.
- Creates an initial Jenkins admin user.
- Installs a curated set of Jenkins plugins.
- Adds GitHub and Docker Hub credentials to Jenkins.
- Creates multibranch seed jobs using Job DSL.
- Installs common build tooling (Docker, Terraform, Packer, Node.js, Python + yamllint).

## Repository layout

- `packer/jenkins-ami.pkr.hcl` – main Packer template and AWS builder config.
- `setup.sh` – instance bootstrap script run by Packer.
- `plugins.txt` – Jenkins plugins installed via CLI.
- `seed-job.groovy` – Job DSL script that creates multibranch pipelines.
- `jenkins-credentials/*.xml` – credential templates populated via `envsubst`.
- `Jenkinsfile` – CI pipeline for commit lint + `packer validate`.

## Prerequisites

Install and configure the following on your local machine or CI runner:

- AWS CLI (authenticated to the target account)
- HashiCorp Packer (v1.7+ recommended)
- Access to an AWS subnet where the build instance can launch
- Jenkins admin credentials and service credentials (GitHub + Docker Hub)

> **Important:** `packer/jenkins-ami.pkr.hcl` currently contains a hard-coded `subnet_id`. Update it for your AWS environment before running a build.

## Configure variables

Create a `packer/jenkins-ami.auto.pkrvars.hcl` file (this is ignored by git) with values for all required variables:

```hcl
JENKINS_ADMIN_USERNAME = "admin"
JENKINS_ADMIN_PASSWORD = "change-me"

GITHUB_USERNAME = "your-github-user"
GITHUB_PASSWORD = "your-github-token-or-password"

DOCKERHUB_USERNAME = "your-dockerhub-user"
DOCKERHUB_PASSWORD = "your-dockerhub-token-or-password"

source_ami = "ami-xxxxxxxxxxxxxxxxx"
```

## Build workflow

Run from the repository root:

```bash
cd packer
packer init jenkins-ami.pkr.hcl
packer fmt -check jenkins-ami.pkr.hcl
packer validate -var-file=jenkins-ami.auto.pkrvars.hcl jenkins-ami.pkr.hcl
packer build -var-file=jenkins-ami.auto.pkrvars.hcl jenkins-ami.pkr.hcl
```

On success, Packer outputs the new AMI ID.

## What gets configured in Jenkins

The provisioner script (`setup.sh`) performs the following high-level actions:

1. Installs Jenkins, Java, and supporting packages.
2. Injects `init.groovy.d` script to create admin user and security setup.
3. Installs plugins listed in `plugins.txt`.
4. Uploads credentials from templated XML files (`github-token`, `docker-hub-credentials`).
5. Executes `seed-job.groovy` to create multibranch pipeline jobs.

## CI pipeline behavior

`Jenkinsfile` in this repo currently:

- Checks out source and captures commit SHA.
- Lints commit messages against Conventional Commits.
- Runs `packer init` + `packer validate`.
- Publishes GitHub commit status (`success`/`failure`) via API.

## Notes and security considerations

- Keep secrets only in `.pkrvars.hcl` files or your CI secret store.
- Never commit real credentials.
- XML credential templates in `jenkins-credentials/` use environment variable placeholders and are rendered at build time.
- Review `seed-job.groovy` repository/org names before production use.

## Troubleshooting

- **`packer validate` fails for missing variables**: ensure all required variables are present in your var-file.
- **Build instance cannot launch**: verify subnet, IAM permissions, and regional AMI ID.
- **Plugins fail to install**: ensure the instance has outbound internet access to Jenkins update centers.
- **Credentials not created**: check that `envsubst` rendered `/tmp/final-*.xml` correctly during provisioning.

---

If you want, I can also add:
- a sample `jenkins-ami.auto.pkrvars.hcl.example`,
- parameterization for region/subnet/instance type,
- and a GitHub Actions workflow that runs `packer fmt` + `packer validate` automatically.
