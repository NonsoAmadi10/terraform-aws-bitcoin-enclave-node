# Bitcoin Enclave Node: A Reference Implementation

This project provides an opinionated, security-first reference implementation for deploying secure Bitcoin infrastructure on AWS using confidential computing. It uses Terraform and AWS Nitro Enclaves to demonstrate how to build infrastructure that is confidential, verifiable, and reproducible.

Think of this repository not as a generic "Bitcoin on AWS" module, but as **executable documentation** for deploying applications under adversarial assumptions.

## 1. Architectural Philosophy

This module is intentionally minimal to demonstrate a set of core infrastructure primitives.

#### What This Module IS:
*   **An Opinionated Reference:** It demonstrates a specific, secure-by-default approach to deploying sensitive workloads.
*   **A Demonstration of Primitives:** It focuses on isolating execution, ensuring key confidentiality, enabling reproducible builds, and facilitating cryptographic attestation.
*   **A Foundation:** It serves as a starting point that can be extended for more complex applications like Arkade nodes or Application Specific Processors (ASPs).

#### What This Module IS NOT:
*   **A Generic "Kitchen Sink" Module:** It avoids excessive toggles and configurations to maintain clarity and focus.
*   **A Wrapper around EC2 + Docker:** The value is in the orchestration of Nitro Enclaves, the attestation flow, and the secure bootstrapping process.
*   **A Full Application:** The included application is a simple "signing" service to demonstrate the architecture, not a production-ready Bitcoin application.

## 2. Threat Model

The primary goal of this architecture is to protect against a compromised cloud environment, whether due to a malicious insider at the cloud provider or an attacker who has gained administrative access to the parent EC2 instance.

#### What This Architecture Protects Against:
*   **Host OS Inspection:** An attacker with root access to the parent EC2 instance cannot inspect the memory or state of the running enclave.
*   **Compromised Storage:** An attacker cannot tamper with the enclave application at rest, as its integrity is verified via a cryptographic hash (PCR0) before launch.
*   **Lack of Confidentiality:** Sensitive data (e.g., signing keys) remains encrypted and isolated within the enclave's memory, inaccessible to the parent OS or the cloud provider.
*   **Configuration Drift:** The entire infrastructure is defined as code, ensuring that deployments are reproducible and auditable.

#### What This Architecture Does NOT Protect Against:
*   **Application-Level Bugs:** A vulnerability within the enclave application code itself can still be exploited.
*   **Side-Channel Attacks:** While Nitro Enclaves provide strong isolation, sophisticated side-channel attacks are a theoretical risk for any co-located compute environment.
*   **Insecure Key Provisioning:** If the initial secret provisioning process is compromised (e.g., a weak KMS policy or a compromised CI/CD pipeline), the enclave can be provisioned with malicious data.
*   **Denial of Service:** An attacker with control of the parent instance can terminate the enclave, leading to a denial of service.

## 3. Core Architecture

The architecture partitions the application into two main components: a trusted **Enclave Application** that runs inside a Nitro Enclave, and an untrusted **Parent Application** that runs on the host EC2 instance.

```
+-------------------------------------------------------------------+
| AWS Cloud                                                         |
|                                                                   |
|   +-----------------------------------------------------------+   |
|   | Parent EC2 Instance (Nitro-based)                         |   |
|   |                                                           |   |
|   |   +-----------------------+       (VSOCK)       +-----------------------+   |
|   |   | Parent Application    |<====================>| Enclave Application   |   |
|   |   | (main.go)             |                     | (enclave/main.go)     |   |
|   |   |                       |                     |                       |   |
|   |   | - Launches Enclave    |                     | - Holds private keys  |   |
|   |   | - Verifies Attestation|                     | - Performs signing    |   |
|   |   | - Manages Networking  |                     |                       |   |
|   |   +-----------------------+                     +-----------------------+   |
|   |                                                 | AWS Nitro Enclave     |   |
|   |                                                 | (Isolated VM)         |   |
|   |                                                 +-----------------------+   |
|   |                                                           |   |
|   +-----------------------------------------------------------+   |
|                                                                   |
+-------------------------------------------------------------------+
```

### Core Components and Application Separation

This project utilizes two distinct Go applications, each named `main.go`, but located in different contexts and serving different purposes:

*   **Parent Application (`enclave_app/src/main.go`):**
    *   **Runs On:** The main EC2 instance (host).
    *   **Role:** This is the orchestrator or "broker." It's responsible for managing the enclave's lifecycle (launching, monitoring, terminating), performing runtime attestation verification, handling external network requests, and securely communicating with the Enclave Application over VSOCK.
*   **Enclave Application (`enclave_app/src/enclave/main.go`):**
    *   **Runs In:** The isolated AWS Nitro Enclave.
    *   **Role:** This is the secure, sensitive part of the workload. It performs critical operations (like cryptographic signing) that require strong isolation. It has no direct access to the network or persistent storage, communicating solely via VSOCK with the Parent Application. This is the code that gets packaged into the Enclave Image File (EIF) and whose integrity is cryptographically measured.

This clear separation is fundamental to the security model of AWS Nitro Enclaves, ensuring that the most sensitive logic is executed in a highly protected environment.

### Attestation and Bootstrapping Flow

The process of launching and verifying the enclave is fully automated:

1.  **Terraform Provisions Host:** Terraform creates the Nitro-enabled EC2 instance, IAM roles, and security groups.
2.  **`user_data` Initializes Host:** On boot, the `user_data` script installs Go, Docker, the Nitro Enclaves CLI, and clones the application git repository.
3.  **`setup_enclave.sh` Builds EIF and Parent App:** The `user_data` script then executes `setup_enclave.sh`, which:
    *   Builds the Go parent application (`enclave_app/src/main.go`) into an executable binary.
    *   Builds the Docker image for the enclave application (`enclave_app/src/enclave/main.go`).
    *   Converts the image into an Enclave Image File (`.eif`).
    *   **Verifies EIF Integrity:** It calculates the PCR0 measurement (a cryptographic hash) of the newly built `.eif` and compares it against the `expected_measurement` provided to Terraform. If they do not match, the script fails, preventing the enclave from starting.
4.  **`systemd` Starts Parent App:** If the EIF is verified, `setup_enclave.sh` sets up and starts a `systemd` service for the parent application (the compiled Go binary).
5.  **Parent App Launches Enclave:** The parent application (the compiled Go binary) then formally launches the enclave using `nitro-cli`.
6.  **Runtime Attestation:** After launch, the parent application requests a signed **attestation document** from the running enclave. It verifies that the PCR0 measurement inside this document also matches the `expected_measurement`.
7.  **Secret Provisioning (Optional):** Only after successful runtime attestation does the parent application proceed. If a `kms_key_arn` is provided, it can now use KMS to decrypt secrets and securely pass them to the enclave over the encrypted VSOCK channel.

## 4. How to Use

This module can be used by following the example provided in the `examples/minimal` directory.

### Step 1: Configure the Example

Navigate to `examples/minimal/main.tf` and configure the module block:

```terraform
module "enclave_node" {
  source = "NonsoAmadi10/bitcoin-enclave-node/aws"

  aws_region = "us-east-1"
  
  # Replace with a PUBLIC git repository containing your enclave_app
  git_repository_url = "https://github.com/your-username/your-enclave-app-repo.git"

  # Restrict SSH ingress to trusted IP ranges. Empty list disables SSH ingress.
  allowed_cidrs = ["203.0.113.10/32"]
  
  # See the PCR0 Measurement Workflow below
  expected_measurement = "your-eif-pcr0-measurement-hash-goes-here"
}
```

### Step 2: The PCR0 Measurement Workflow

The `expected_measurement` variable is the key to ensuring the integrity of your enclave. Here is the workflow to set it correctly:

1.  **First Deployment (Leave Blank):** On your very first deployment, comment out or leave `expected_measurement` as an empty string.
2.  **Apply Terraform:** Run `terraform init` and `terraform apply`. This will build the EC2 instance.
3.  **Find the PCR0 Measurement:** The `user_data` script will build the `.eif` and calculate its PCR0 hash. You can find this value in the EC2 instance logs (either via the AWS Console's "Instance Settings -> Get instance screenshot" or by SSHing into the instance and checking the systemd logs for `enclave-broker.service`). Look for a line like this:
    ```
    Actual EIF PCR0 Measurement: 9a8b...
    ```
4.  **Update and Redeploy:** Copy this hash value and paste it into the `expected_measurement` variable in your `.tf` file.
5.  **Subsequent Deployments:** From now on, every time Terraform runs, the `setup_enclave.sh` script will verify that the newly built EIF has this exact measurement. If it ever changes, the setup will fail, protecting you from running untrusted code.

## 5. Module Inputs and Outputs

### Inputs

| Name                   | Description                                                                                         | Type          | Default       |
| ---------------------- | --------------------------------------------------------------------------------------------------- | ------------- | ------------- |
| `aws_region`           | The AWS region to deploy resources in.                                                              | `string`      | (required)    |
| `name_prefix`          | A prefix used for all created resources to ensure unique names.                                     | `string`      | `"btc-enclave"` |
| `git_repository_url`   | The URL of the git repository to clone onto the instance. Must contain the `enclave_app` directory.   | `string`      | (required)    |
| `instance_type`        | The EC2 instance type for the enclave host. Must be Nitro Enclaves compatible.                        | `string`      | `"m5.xlarge"` |
| `allowed_cidrs`        | A list of CIDR blocks allowed to SSH to the host. Empty list disables SSH ingress.                 | `list(string)`| `[]`          |
| `expected_measurement` | The expected PCR0 measurement of the enclave image. Used for integrity verification.                | `string`      | `""`          |
| `kms_key_arn`          | Optional: The ARN of the KMS key to use for secret unwrapping within the enclave.                     | `string`      | `null`        |
| `log_sink`             | Optional: The destination for logs (e.g., CloudWatch Log Group ARN).                                | `string`      | `null`        |
| `enclave_cpu_count`    | The number of vCPUs to allocate to the Nitro Enclave.                                               | `number`      | `1`           |
| `enclave_memory_mib`   | The amount of memory (in MiB) to allocate to the Nitro Enclave.                                     | `number`      | `256`         |
| `vpc_id`               | Optional: The ID of the VPC to deploy the host in. If not provided, the default VPC is used.          | `string`      | `null`        |
| `subnet_id`            | Optional: The ID of the subnet to deploy the host in. If not provided, a default subnet is used.    | `string`      | `null`        |

### Outputs

| Name                   | Description                                                                               |
| ---------------------- | ----------------------------------------------------------------------------------------- |
| `instance_id`          | The ID of the EC2 instance hosting the enclave.                                           |
| `instance_public_ip`   | Public IP address of the EC2 instance.                                                    |
| `enclave_measurement`  | The expected PCR0 measurement of the enclave image that the system will verify against.   |

## 6. Future Improvements

This reference implementation provides a strong foundation. However, to make it truly production-ready, several areas could be improved and hardened:

*   **Full KMS Integration:** The current KMS integration is a placeholder. A full implementation would involve the parent application passing only encrypted ciphertext to the enclave, with the enclave being the only entity capable of calling `kms:Decrypt` after successful attestation.

*   **Robust Communication Protocol:** The current communication between the parent and enclave is a simple echo service. This could be developed into a more robust API using a standard like JSON-RPC over VSOCK to handle specific commands (e.g., `sign_transaction`, `get_public_key`).

*   **CI/CD for EIF Builds:** For maximum security and reproducibility, the Enclave Image File (EIF) should be built in a trusted CI/CD pipeline, not on the host at boot time. The resulting EIF and its PCR0 measurement should be stored in a secure registry (like ECR), and the EC2 instance should pull this pre-verified artifact.

*   **Enhanced Logging and Monitoring:** The parent application could be configured to emit structured (JSON) logs to CloudWatch Logs. Custom metrics (e.g., `attestation_failures`, `signing_operations_per_second`) could also be published to CloudWatch to provide better observability into the health and security of the system.

*   **Host and Network Hardening:** The host environment could be further secured by using a minimal, hardened OS (like Bottlerocket) and by restricting security group egress rules to only the essential endpoints (e.g., AWS KMS).

## 7. Contributing

Contributions are welcome! This project is intended as a learning tool and a foundation for more complex systems. If you have ideas for improvements or find a bug, please feel free to:

1.  **Open an Issue:** Discuss the change you would like to make.
2.  **Submit a Pull Request:** Fork the repository, make your changes, and open a pull request for review.

Please make sure to update tests as appropriate and ensure the CI pipeline passes.

## 8. CI/CD

The repository includes a GitHub Actions workflow (`.github/workflows/ci.yml`) that runs:

- `terraform fmt -check -recursive`
- `terraform init -backend=false` and `terraform validate` in `examples/minimal`
- `tflint` on the example module tree
- `go vet` and `go test` for the enclave app

This default CI does not require AWS credentials.  
If you later add deployment workflows (`terraform plan/apply`), prefer OIDC-based short-lived credentials and scope IAM permissions to least privilege.
