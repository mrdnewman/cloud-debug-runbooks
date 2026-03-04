# EC2 IMPORT INTO TERRAFORM (FULL STEP-BY-STEP)

1) In your Terraform code, create the resource block (placeholder):
   resource "aws_instance" "web" {}

2) Initialize Terraform (providers + backend):
   terraform init

3) Find the EC2 instance ID in AWS (filter by Name tag if possible):
   aws ec2 describe-instances --filters "Name=tag:Name,Values=web*" --query "Reservations[*].Instances[*].InstanceId" --output text
   # (if you can’t filter) aws ec2 describe-instances --query "Reservations[*].Instances[*].InstanceId" --output text

4) Import that instance into Terraform state:
   terraform import aws_instance.web i-XXXXXXXXXXXXXXX

5) Inspect what Terraform imported (this tells you the real values you must match):
   terraform state show aws_instance.web

6) Edit the resource block to match the imported attributes (minimum: ami, instance_type, subnet_id, vpc_security_group_ids, tags):
   resource "aws_instance" "web" {
     ami                    = "ami-..."
     instance_type          = "t3.micro"
     subnet_id              = "subnet-..."
     vpc_security_group_ids = ["sg-..."]
     tags = { Name = "web" }
   }

7) Run plan to confirm Terraform won’t try to destroy/recreate unexpectedly:
   terraform plan

8) If plan is clean (or only intended changes), apply:
   terraform apply
