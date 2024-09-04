### Terraform code for AWS Security Group, allowing you to define the rules for inbound and outbound traffic based on the protocol (TCP/UDP), port range, and source/destination IP address

```hcl
# Create security group to allow port 80, 443 and 22 (SSH port) from public (0.0.0.0/0)
resource "aws_security_group" "my_sg" {
  vpc_id = aws_vpc.my_vpc.id
  name = "my_sg"
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "my_sg"
  }
}
```