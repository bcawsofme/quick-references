# Terraform and OpenTofu Quick Reference Guide

## Project Setup
```bash
terraform version
terraform init
terraform init -upgrade
terraform fmt
terraform fmt -recursive
terraform validate
```

## Plan and Apply
```bash
terraform plan
terraform plan -out=tfplan
terraform apply
terraform apply tfplan
terraform apply -auto-approve
terraform destroy
terraform destroy -auto-approve
```

## Variables
```bash
terraform plan -var="env=dev"
terraform apply -var-file=dev.tfvars
terraform console
terraform output
terraform output <name>
terraform output -json
```

## State
```bash
terraform state list
terraform state show <address>
terraform state mv <source> <destination>
terraform state rm <address>
terraform state pull
terraform refresh
```

## Import and Drift
```bash
terraform import <address> <id>
terraform plan -refresh-only
terraform apply -refresh-only
terraform taint <address>
terraform untaint <address>
terraform replace -target=<address>
```

## Workspaces
```bash
terraform workspace list
terraform workspace show
terraform workspace new <name>
terraform workspace select <name>
terraform workspace delete <name>
```

## Modules and Providers
```bash
terraform providers
terraform providers lock
terraform get
terraform get -update
terraform graph
```

## Targeting and Debugging
```bash
terraform plan -target=<address>
terraform apply -target=<address>
terraform plan -destroy
terraform plan -detailed-exitcode
TF_LOG=debug terraform plan
TF_LOG=trace terraform apply
```

## OpenTofu Equivalents
```bash
tofu init
tofu fmt -recursive
tofu validate
tofu plan
tofu apply
tofu destroy
tofu state list
tofu import <address> <id>
```
