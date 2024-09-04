### Here is an example of a Terraform code that deploys EC2 instance

# AMI lookup for latest Ubuntu AMI

```hcl
# Terraform code for getting the latest Ubuntu AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
  owners = ["099720109477"] # Canonical
}
```

# Terraform code to create EC2 instance with 30GB EBS Volume

```hcl
# Terraform code for EC2 On Demand instance
resource "aws_instance" "my_instance" {
  ami           = "ami-0864c7b539c74f162"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public_subnet[0].id # If only 1 public sunbet is there then use this "aws_subnet.public.id"

  root_block_device {
    volume_size = 30
  }

  tags = {
    Name = "My EC2 Instance"
  }
}
```

# Terraform code to create EC2 Spot instance

```hcl
# Terraform code for AMI lookup for latest Amazon Linux AMI
data "aws_ami" "amzn-linux-2023-ami" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }
}

# Terraform code for EC2 Spot instance
resource "aws_instance" "my_spot_instance" {
  ami = data.aws_ami.amzn-linux-2023-ami.id
  instance_market_options {
    spot_options {
      max_price = 0.0031
    }
  }
  instance_type = "t4g.nano"
  tags = {
    Name = "test-spot"
  }
}
```
### Terraform code to generate SSH Key for Ec2 instance

```hcl
# Terraform code to generate RSA 2048 bit key
resource "tls_private_key" "private_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

# Terraform code to create Public Key pair
resource "aws_key_pair" "public_key_pair" {
  key_name   = "my_ssh_key"
  public_key = tls_private_key.private_key.public_key_openssh
}

# Terraform code to create local SSH private key
resource "local_file" "ssh_key" {
  filename        = "my_ec2_key.pem"
  content         = tls_private_key.private_key.private_key_pem
  file_permission = "0400"
}

```