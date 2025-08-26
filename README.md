# Terraform provider Nexus (Fork with Write-Only Password Support)

![codeql workflow](https://github.com/yourusername/terraform-provider-nexus/actions/workflows/codeql-analysis.yml/badge.svg)
[![Contributor Covenant](https://img.shields.io/badge/Contributor%20Covenant-2.1-4baaaa.svg)](CODE_OF_CONDUCT.md)
[![Go Report Card](https://goreportcard.com/badge/github.com/yourusername/terraform-provider-nexus)](https://goreportcard.com/report/github.com/yourusername/terraform-provider-nexus)

> **Note**: This is a fork of [datadrivers/terraform-provider-nexus v2.6.0](https://github.com/datadrivers/terraform-provider-nexus) with added support for write-only passwords. See [Pull Request #538](https://github.com/datadrivers/terraform-provider-nexus/pull/538) for upstream integration status.

- [Terraform provider Nexus (Fork)](#terraform-provider-nexus-fork-with-write-only-password-support)
  - [Fork Changes](#fork-changes)
  - [Installation](#installation)
  - [Write-Only Password Feature](#write-only-password-feature)
  - [Migration from Original Provider](#migration-from-original-provider)
  - [Introduction](#introduction)
  - [Usage](#usage)
    - [Provider config](#provider-config)
  - [Development](#development)
    - [Build](#build)
    - [Testing](#testing)
      - [To debug tests](#to-debug-tests)
    - [Create documentation](#create-documentation)
  - [Author](#author)

## Fork Changes

This fork adds secure password management capabilities to prevent passwords from being stored in Terraform state files.

### New in this Fork (v2.6.0-writeonly.x):
- **Write-only password support** for `nexus_security_user` resource
- **Ephemeral resource compatibility** (works with `random_password`, external secrets)
- **Automatic password rotation** support
- **Full backward compatibility** with existing configurations


## Installation

Add this provider to your Terraform configuration:

```hcl
terraform {
  required_providers {
    nexus = {
      source  = "defimenko-ops/nexus"  # Your published provider
      version = "~> 1.0.0"
    }
  }
}
```

## Write-Only Password Feature

### Secure Password Management

```hcl
resource "nexus_security_user" "secure_user" {
  userid              = "secure-user"
  firstname           = "Secure"
  lastname            = "User"
  email               = "secure@example.com"
  password_wo         = "secret-password"    # NOT stored in state
  password_wo_version = 1                    # Only version tracked
  status              = "active"
  roles               = ["nx-admin"]
}
```

### With Random Passwords

```hcl
ephemeral "random_password" "password" {
  length           = 16
}

resource "nexus_security_user" "random_user" {
  userid              = "random-user"
  firstname           = "Random"
  lastname            = "User"
  email               = "random@example.com"
  password_wo         = ephemeral.random_password.password.result  # Ephemeral
  password_wo_version = 1
  status              = "active"
  roles               = ["nx-developer"]
}
```

### Password Updates

```hcl
# To update password, increment the version:
resource "nexus_security_user" "secure_user" {
  userid              = "secure-user"
  password_wo         = "new-secret-password"
  password_wo_version = 2                    # Triggers password update
  # ... other fields unchanged
}
```

### External Secret Management

```hcl
data "aws_secretsmanager_secret_version" "nexus_password" {
  secret_id = "prod/nexus/admin-password"
}

resource "nexus_security_user" "admin" {
  userid              = "admin-secure"
  password_wo         = jsondecode(data.aws_secretsmanager_secret_version.nexus_password.secret_string)["password"]
  password_wo_version = jsondecode(data.aws_secretsmanager_secret_version.nexus_password.secret_string)["version"]
  # ... other fields
}
```

## Migration from Original Provider

### Option 1: Keep existing configuration
```hcl
# This continues to work (backward compatible)
resource "nexus_security_user" "legacy" {
  userid   = "legacy-user"
  password = "still-works"  # Still stored in state
  # ... other fields
}
```

### Option 2: Migrate to secure approach
```hcl
# Change to write-only password
resource "nexus_security_user" "migrated" {
  userid              = "legacy-user"        # Same user ID
  password_wo         = "secure-password"    # New secure field
  password_wo_version = 1                    # Add version tracking
  # ... other fields unchanged
}
```

## Introduction

Terraform provider to configure Sonatype Nexus using its API.

Implemented and tested with Sonatype Nexus `3.80.0` with `java17` and DB `H2`.

## Usage

### Provider config

```hcl
provider "nexus" {
  insecure         = true
  password         = "admin123"
  url              = "https://127.0.0.1:8080"
  username         = "admin"
}
```

Optionally with mTLS if Nexus is deployed behind a reverse proxy:

```hcl
provider "nexus" {
  insecure         = true
  password         = "admin123"
  url              = "https://127.0.0.1:8080"
  username         = "admin"
  client_cert_path = "/path/to/client.crt"
  client_key_path  = "/path/to/client.key"
  root_ca_path     = "/path/to/root_ca.crt"
}
```

Note that the `root_ca_path` should contain ALL certificates required for
communication. It overrides the system CA store, rather than adding to it.

You can point the `root_ca_path` to the system trust store if required, e.g.:

`root_ca_path = "/etc/ssl/certs/ca-certificates.crt"`

## Development

### Build

There is a [makefile](./GNUmakefile) to build the provider and place it in repos root dir.

```sh
make
```

To use the local build version you need tell terraform where to look for it via a terraform config override.

Create `dev.tfrc` in your terraform code folder (f.e. in [dev.tfrc](./examples/local-development/dev.tfrc)):

```hcl
# dev.tfrc
provider_installation {

  # Use /home/developer/tmp/terraform-nexus as an overridden package directory
  # for the datadrivers/nexus provider. This disables the version and checksum
  # verifications for this provider and forces Terraform to look for the
  # nexus provider plugin in the given directory.
  # relative path also works, but no variable or ~ evaluation
  dev_overrides {
    "datadrivers/nexus" = "../../"
  }

  # For all other providers, install them directly from their origin provider
  # registries as normal. If you omit this, Terraform will _only_ use
  # the dev_overrides block, and so no other providers will be available.
  direct {}
}
```

Tell your shell environment to use override file:

```bash
export TF_CLI_CONFIG_FILE=dev.tfrc
```

Now run your terraform commands (`plan` or `apply`), `init` is ***not*** required.

```bash
# start local nexus
make start-services
# run local terraform code
cd examples/local-development
terraform plan
terraform apply
```

### Testing

**NOTE**: For testing Nexus Pro features, place the `license.lic` in `scripts/`.

For testing start a local Docker containers using make

```shell
make start-services
```

This will start a Docker and MinIO containers and expose ports 8081 and 9000.

Now start the tests

```shell
make testacc
```

or skipped tests:

```shell
SKIP_S3_TESTS=true make testacc
SKIP_AZURE_TESTS=true make testacc
SKIP_PRO_TESTS=true make testacc
```

#### To debug tests

Set env variable `TF_LOG=DEBUG` to see additional output.

Use `printState()` function to discover terraform state (and resource props) during test.

Debug configurations are also available for VS Code.

### Create documentation

When creating or updating resources/data resources please make sure to update the examples in the respective folder (`./examples/resources/<name>` for resources, `./examples/data-sources/<name>` for data sources)

Next you can use the following command to generate the terraform documentation from go files

```shell
make docs
```

## Author

[Datadrivers GmbH](https://www.datadrivers.de)
