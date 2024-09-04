### Terraform code for deployming Application load balancer

```hcl
# Terraform code for Public ALB Loadbalancer creation
resource "aws_alb" "elb" {
  name            = "my-elb"
  subnets         = aws_subnet.public.*.id
  security_groups = [aws_security_group.my_sg.id]

  tags = {
    Name        = "my_alb"
  }
}

# Terraform code for ALB target group creation
resource "aws_alb_target_group" "my_tgrp" {
  name                 = "my-tgrp"
  port                 = "80"
  protocol             = "HTTP"
  vpc_id               = aws_vpc.my_vpc.id
  target_type          = "instance"
  deregistration_delay = "30"

  health_check {
    healthy_threshold   = "2"
    interval            = "30"
    protocol            = "HTTP"
    matcher             = "200"
    timeout             = "25"
    path                = "/"
    unhealthy_threshold = "3"
  }
  tags = {
    Name        = "my_tgrp"
  }
}

# Terraform code HTTP Listner, redirect all traffic from the port 80 to 443
resource "aws_alb_listener" "http" {
  load_balancer_arn = aws_alb.elb.id
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# Terraform code to create HTTPS listner for ELB
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_alb.elb.id // Amazon Resource Name (ARN) of the load balancer
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-2016-08"
  certificate_arn   = var.ssl_cert

  // By default, forward to targetgroup
  default_action {
    type             = "forward"
    target_group_arn = aws_alb_target_group.my_tgrp.arn
  }
}
```