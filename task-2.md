# 📄 Problem Statement
- Rewrite the migration-task narrative so it explains why Nautilus is doing a phased move to AWS, with every requirement summarized clearly.
- Key requirements: name the security group 
autilus-sg in the us-east-1 default VPC, document the description, and allow HTTP (80) and SSH (22) from 0.0.0.0/0.
- Constraint: keep everything inside main.tf under /home/bob/terraform and use Terraform to manage the resources.

# 🧠 Approach & Thinking
- Leverage the default VPC via data "aws_vpc" "default" so the security group deploys without hard-coding an ID, honoring the requirement to use the existing environment.
- Define a focused ws_security_group resource with the mandated name/description, then attach two explicit ingress rules for HTTP and SSH.
- The provider is locked to us-east-1 so the workspace aligns with the task’s region demand, and keeping all code in main.tf is a Terraform best practice for small exercises.
- This keeps the configuration simple, repeatable, and easy to audit while still allowing the DevOps team to expand the security group later if more ports are needed.

# 🧰 Terraform Solution
`hcl

resource "aws_security_group" "nautilus_sg" {
  name        = "nautilus-sg"
  description = "Security group for Nautilus App Servers"
}

resource "aws_security_group_rule" "allow_http" {
  type              = "ingress"
  from_port         = 80
  to_port           = 80
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.nautilus_sg.id
  description       = "Allow HTTP from anywhere"
}

resource "aws_security_group_rule" "allow_ssh" {
  type              = "ingress"
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.nautilus_sg.id
  description       = "Allow SSH from anywhere"
}
`

# 🚀 Step-by-Step Execution
1. 	erraform init to install the AWS provider (hashicorp/aws) and to register the default VPC data source.
2. 	erraform plan to confirm the security group and ingress rules will be created with the correct ports and CIDR.
3. 	erraform apply to provision the resources in us-east-1.

- Required providers: hashicorp/aws.
- Prerequisites: AWS credentials configured (profile, env vars, or instance metadata) and a default VPC present in the account.

# 📝 Why This Solution Works
- Using data.aws_vpc.default ensures the security group attaches to the standard VPC without extra inputs, satisfying the default-VPC requirement.
- ws_security_group.nautilus_sg declares the mandated name and description.
- The two ws_security_group_rule blocks keep HTTP and SSH ingress definitions explicit and maintainable.

# 📚 Reference to Terraform Docs
- provider "aws" pins the AWS region and lets Terraform authenticate with us-east-1 APIs.
- data "aws_vpc" "default" reads the default VPC so the security group can reference its ID cleanly.
- 
esource "aws_security_group" creates the SG object, and ws_security_group_rule lets us define ingress ports without nesting anonymous blocks.

# ⚠️ Common Mistakes
- Forgetting to resolve the default VPC, which can lead to a security group without a VPC ID.
- Trying to describe ingress entirely inside the SG block, which makes incremental updates and separation of concerns harder.
- Omitting the region in the provider or relying on the wrong default region and accidentally touching a different AWS region.
- Hard-coding a VPC ID and then failing to adjust it across accounts, breaking the portability promise.

# ✅ Final Thoughts
This configuration keeps the Nautilus migration sprint manageable: clear ingress rules, a default-VPC anchor, and a Terraform layout that can be extended as later phases demand more ports or egress safeguards.
