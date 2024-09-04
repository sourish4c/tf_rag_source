

### Terraform code for Application Load Balancer

```terraform
resource "aws_lb" "test" {
  name               = "test-lb-tf"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.lb_sg.id]
  subnets            = [for subnet in aws_subnet.public : subnet.id]

  enable_deletion_protection = true

  access_logs {
    bucket  = aws_s3_bucket.lb_logs.id
    prefix  = "test-lb"
    enabled = true
  }

  tags = {
    Environment = "production"
  }
}
```

### Terraform code for Network Load Balancer

```terraform
resource "aws_lb" "test" {
  name               = "test-lb-tf"
  internal           = false
  load_balancer_type = "network"
  subnets            = [for subnet in aws_subnet.public : subnet.id]

  enable_deletion_protection = true

  tags = {
    Environment = "production"
  }
}
```

### Specifying Elastic IPs

```terraform
resource "aws_lb" "example" {
  name               = "example"
  load_balancer_type = "network"

  subnet_mapping {
    subnet_id     = aws_subnet.example1.id
    allocation_id = aws_eip.example1.id
  }

  subnet_mapping {
    subnet_id     = aws_subnet.example2.id
    allocation_id = aws_eip.example2.id
  }
}
```

### Specifying private IP addresses for an internal-facing load balancer

```terraform
resource "aws_lb" "example" {
  name               = "example"
  load_balancer_type = "network"

  subnet_mapping {
    subnet_id            = aws_subnet.example1.id
    private_ipv4_address = "10.0.1.15"
  }

  subnet_mapping {
    subnet_id            = aws_subnet.example2.id
    private_ipv4_address = "10.0.2.15"
  }
}
```

