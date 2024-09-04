### Network Architecture example

In creating network architecture all the resources are deployed in network resources.tf file. Basically, Network architecture includes the below aws services.

- One (1) VPC.
- Two (2) public subnets spread across two availability zones (Web Tier).
- Two (2) private subnets spread across two availability zones (Application Tier).
- One (1) public route table that connects the public subnets to an internet gateway.
- One (1) private route table that will connect the Application Tier private subnets and a NAT gateway.

### Terraform code to create 1 VPC, 2 Public Subnets, 2 Private Subnets, 1 Internet Gateway and 1 NAT Gateway

```hcl
# Create VPC
resource "aws_vpc" "my_vpc" {
  cidr_block = "172.0.0.0/16"
  tags = {
    Name = "my_vpc"
  }
}

# Create public subnets
resource "aws_subnet" "public_subnet" {
  count                   = 2 # Count function to create 2 public subnet resources
  vpc_id                  = aws_vpc.my_vpc.id
  cidr_block              = cidrsubnet(aws_vpc.my_vpc.cidr_block, 8, count.index)
  map_public_ip_on_launch = true
  tags = {
    Name = "public-${count.index + 1}"
  }
}

# Create private subnets
resource "aws_subnet" "private_subnet" {
  count      = 2 # Count function to create 2 private subnet resources
  vpc_id     = aws_vpc.my_vpc.id
  cidr_block = cidrsubnet(aws_vpc.my_vpc.cidr_block, 8, count.index + 2)
  tags = {
    Name = "private-${count.index + 1}"
  }
}

# Create internet gateway
resource "aws_internet_gateway" "my_igw" {
  vpc_id = aws_vpc.my_vpc.id
  tags = {
    Name = "my_igw"
  }
}

# IMPORTANT Creation of aws_eip resource is mandatory for creating nat gateway. Hence resource "aws_eip" has to be generated before generating the resource "aws_nat_gateway"

# Create EIP for AWS NAT gateway. This EIP will be attached to the NAT Gateway
resource "aws_eip" "my_eip" {
  domain = "vpc" # This is the new configuration, it was previously vpc = true
  tags = {
    Name = "my_nat_IP"
  }
}

# Create AWS NAT gateway 
resource "aws_nat_gateway" "my_nat_gateway" {
  allocation_id = aws_eip.my_eip.id
  subnet_id     = aws_subnet.public_subnet[0].id
  tags = {
    Name = "my_nat_gw"
  }
}
```

### Terraform code to create route tables and route table association with public and private subnets

```hcl
# Routing table for private subnet
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.my_vpc.id
  tags = {
    Name = "my-private-rt"
  }
}

# Routing table for public subnet
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.my_vpc.id
  tags = {
    Name = "my-public-rt"
  }
}

# Route for public subnet via internet gateway
resource "aws_route" "public_internet_gateway" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.my_igw.id
}

# Route for private subnet via nat gateway
resource "aws_route" "private_nat_gateway" {
  route_table_id         = aws_route_table.private.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.my_nat_gateway.id
}

# Route table association for public subnets
resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public_subnet)
  subnet_id      = element(aws_subnet.public_subnet.*.id, count.index)
  route_table_id = aws_route_table.public.id
}

# Route table association for private subnets
resource "aws_route_table_association" "private" {
  count          = length(aws_subnet.private_subnet)
  subnet_id      = element(aws_subnet.private_subnet.*.id, count.index)
  route_table_id = aws_route_table.private.id
  depends_on     = [ aws_nat_gateway.my_nat_gateway ]
}
```