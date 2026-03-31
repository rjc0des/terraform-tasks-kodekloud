# Problem Statement
- A controlled AWS migration is unfolding in bite-sized phases, so the Nautilus DevOps team needs Terraform to own the creation of an RSA key pair for future instances without disrupting other work.  
- Requirements: a key named `devops-kp`, RSA algorithm, private key persisted to `/home/bob/devops-kp.pem`, and everything declared in `/home/bob/terraform/main.tf` (no extra `.tf` files).  
- Constraint: the private key must be generated and stored locally (not manually uploaded) so every phase can reference a predictable credential automatically.

# Approach & Thinking
- Break the flow into generating the RSA key material, using Terraform to register the public portion with AWS, and persisting the private key file locally. Separating these steps keeps the migration modular—Terraform manages the credential lifecycle while the pipeline focuses on provisioning compute in later phases.  
- Alternative: manually create the key with `ssh-keygen` and only import the public key via `aws_key_pair`. That adds extra operational steps and breaks the “pure Terraform” requirement; it also wouldn’t guarantee Terraform can recreate or rotate the private key on future runs.  
- By generating the TLS key on the fly and saving it with `local_file`, the plan stays idempotent and self-contained, which is better for incremental migrations where repeatable automation matters.

# Terraform Solution
```hcl
resource "tls_private_key" "devops_key" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "devops_kp" {
  key_name   = "devops-kp"
  public_key = tls_private_key.devops_key.public_key_openssh
}

resource "local_file" "save_to_local" {
  content       = tls_private_key.devops_key.private_key_pem
  filename      = "/home/bob/devops-kp.pem"
  file_permission = "0600"
}
```

# Step-by-Step Execution
1. `terraform init` in `/home/bob/terraform` to install the `aws`, `tls`, and `local` providers.  
2. `terraform plan` to verify Terraform will create the key pair and write `/home/bob/devops-kp.pem`.  
3. `terraform apply` to generate the RSA key, register the AWS key pair, and persist the private key file.  

Prerequisites: AWS credentials configured in the environment or shared config, write permissions for `/home/bob/devops-kp.pem`, and Terraform installed in a version that supports `required_providers` blocks.

# Why This Solution Works
- `tls_private_key` meets the RSA requirement and ensures Terraform fully controls key generation.  
- `aws_key_pair` publishes the public key to AWS under the mandated name, so future instances can reference it.  
- `local_file` persists the private key exactly where the migration plan expects it, with secure permissions in place. Each block has a clear, reproducible responsibility aligned with the migration constraint.

# Reference to Terraform Docs
- `tls_private_key`: generates local crypto material so Terraform can own key rotation without external tooling.  
- `aws_key_pair`: uploads a public key to AWS EC2, letting compute resources (and automation) reference `devops-kp`.  
- `local_file`: writes files on the controller machine, which is essential for ensuring `/home/bob/devops-kp.pem` exists for later stages.

# Common Mistakes
- Generating the key outside Terraform (e.g., `ssh-keygen`) and hard-coding the public key defeats the self-contained requirement.  
- Forgetting to restrict `/home/bob/devops-kp.pem`’s permissions exposes the private key.  
- Omitting the `local_file` resource would leave the private key only in state, making it unusable for future manual steps.  
- Using a different key name or algorithm would break dependents expecting `devops-kp`/RSA for the incremental migration rollout.

# Final Thoughts
- This Terraform-first key generation keeps each migration phase repeatable and auditable—Terraform generates the full credential set, publishes the public half to AWS, and stores the private half locally with secure permissions. It’s a solid pattern whenever you want infrastructure code to own both cloud and controller-side artifacts without manual intervention.
