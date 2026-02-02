---
date: 2026-01-18
title: Terraform subnet splitting
categories:
  - Terraform
tags:
  - Terraform
  - cidrsubnet
slug: terraform-subnet-splitting
---

# Subnet Splitting in Terraform using `cidrsubnet`

When creating VPCs in AWS, one of the most common tasks is splitting your VPC CIDR block into multiple subnets across different availability zones. This is something I do frequently but always forget the exact approach, so I'm documenting it here for future reference.

## Understanding `cidrsubnet`

Terraform's `cidrsubnet` function is the key to this approach. It has the following signature:

```
cidrsubnet(prefix, newbits, netnum)
```

- **prefix**: The base CIDR block (e.g., `10.0.0.0/16`)
- **newbits**: Number of additional bits to add to the prefix length
- **netnum**: The subnet number (0-indexed) to create

The function creates `2^newbits` possible subnets, and `netnum` selects which one.

## Two-Level Splitting Approach

My approach uses a two-level hierarchy:

1. **First level**: Split the VPC CIDR into three subnet types (web, backend, datastore)
2. **Second level**: Split each block per availability zone

### First Level: Web, Backend, and Datastore

The three subnet types serve different purposes:
- **Web subnets**: Publicly available services (with Internet Gateway)
- **Backend subnets**: Other services that need outbound internet access (with NAT Gateway)
- **Datastore subnets**: Data services that don't need internet access (no NAT Gateway)

```hcl
vpc_web_subnet_block = cidrsubnet(var.vpc_cidr, 2, 0)
vpc_backend_subnet_block = cidrsubnet(var.vpc_cidr, 2, 1)
vpc_datastore_subnet_block = cidrsubnet(var.vpc_cidr, 2, 2)
```

Using `newbits = 2` creates 4 possible subnets (2^2 = 4), and we use 3 of them:
- `netnum = 0` → Web subnets (publicly accessible)
- `netnum = 1` → Backend subnets (with NAT Gateway)
- `netnum = 2` → Datastore subnets (no NAT Gateway)
- `netnum = 3` → Unused (available for future expansion)

**Example**: If `vpc_cidr = "10.0.0.0/16"`:
- Web block: `10.0.0.0/18` (first quarter)
- Backend block: `10.0.64.0/18` (second quarter)
- Datastore block: `10.0.128.0/18` (third quarter)

### Second Level: Per Availability Zone

The number of bits needed depends on how many availability zones you're using:

```hcl
number_of_zones = var.env_type == "production" ? 3 : 2
zone_bits = local.number_of_zones == 2 ? 1 : 2
```

- **2 AZs**: Use `zone_bits = 1` (creates 2^1 = 2 subnets)
- **3 AZs**: Use `zone_bits = 2` (creates 2^2 = 4 subnets, but we only use 3)

Then split each block per availability zone:

```hcl
web_subnets = {
  for idx, az in local.azs :
    az => {
      az = az
      cidr = cidrsubnet(local.vpc_web_subnet_block, local.zone_bits, idx)
    }
}

backend_subnets = {
  for idx, az in local.azs :
    az => {
      az = az
      cidr = cidrsubnet(local.vpc_backend_subnet_block, local.zone_bits, idx)
    }
}

datastore_subnets = {
  for idx, az in local.azs :
    az => {
      az = az
      cidr = cidrsubnet(local.vpc_datastore_subnet_block, local.zone_bits, idx)
    }
}
```

**Example with 2 AZs** (`zone_bits = 1`):
- Web block `10.0.0.0/18` splits into:
  - AZ 1: `10.0.0.0/19` (idx=0)
  - AZ 2: `10.0.32.0/19` (idx=1)
- Backend block `10.0.64.0/18` splits into:
  - AZ 1: `10.0.64.0/19` (idx=0)
  - AZ 2: `10.0.96.0/19` (idx=1)
- Datastore block `10.0.128.0/18` splits into:
  - AZ 1: `10.0.128.0/19` (idx=0)
  - AZ 2: `10.0.160.0/19` (idx=1)

**Example with 3 AZs** (`zone_bits = 2`):
- Web block `10.0.0.0/18` splits into:
  - AZ 1: `10.0.0.0/20` (idx=0)
  - AZ 2: `10.0.16.0/20` (idx=1)
  - AZ 3: `10.0.32.0/20` (idx=2)
- Backend block `10.0.64.0/18` splits into:
  - AZ 1: `10.0.64.0/20` (idx=0)
  - AZ 2: `10.0.80.0/20` (idx=1)
  - AZ 3: `10.0.96.0/20` (idx=2)
- Datastore block `10.0.128.0/18` splits into:
  - AZ 1: `10.0.128.0/20` (idx=0)
  - AZ 2: `10.0.144.0/20` (idx=1)
  - AZ 3: `10.0.160.0/20` (idx=2)
  - (idx=3 is unused, but that's okay)

## Complete Example

For a VPC with CIDR `10.0.0.0/16` and 2 availability zones:

```
VPC: 10.0.0.0/16
├── Web: 10.0.0.0/18 (Internet Gateway)
│   ├── AZ-1: 10.0.0.0/19
│   └── AZ-2: 10.0.32.0/19
├── Backend: 10.0.64.0/18 (NAT Gateway)
│   ├── AZ-1: 10.0.64.0/19
│   └── AZ-2: 10.0.96.0/19
└── Datastore: 10.0.128.0/18 (No NAT Gateway)
    ├── AZ-1: 10.0.128.0/19
    └── AZ-2: 10.0.160.0/19
```

For 3 availability zones:

```
VPC: 10.0.0.0/16
├── Web: 10.0.0.0/18 (Internet Gateway)
│   ├── AZ-1: 10.0.0.0/20
│   ├── AZ-2: 10.0.16.0/20
│   └── AZ-3: 10.0.32.0/20
├── Backend: 10.0.64.0/18 (NAT Gateway)
│   ├── AZ-1: 10.0.64.0/20
│   ├── AZ-2: 10.0.80.0/20
│   └── AZ-3: 10.0.96.0/20
└── Datastore: 10.0.128.0/18 (No NAT Gateway)
    ├── AZ-1: 10.0.128.0/20
    ├── AZ-2: 10.0.144.0/20
    └── AZ-3: 10.0.160.0/20
```

## Routing Considerations

Each subnet type has different routing requirements:

- **Web subnets**: Route table with Internet Gateway for public access
- **Backend subnets**: Route table with NAT Gateway for outbound internet access
- **Datastore subnets**: Route table with no internet gateway or NAT Gateway (isolated)

This separation ensures that:
- Public-facing services are in web subnets with direct internet access
- Backend services can reach the internet for updates, API calls, etc., via NAT Gateway
- Datastore services remain isolated with no internet access, improving security

## Why This Approach Works

1. **Predictable**: The hierarchy is clear and easy to understand
2. **Scalable**: Easy to add more subnet types (e.g., EKS subnets) using the same pattern
3. **Balanced**: Each subnet type gets equal address space (one-third of the VPC)
4. **Flexible**: Works for both 2 and 3 AZ scenarios
5. **Secure**: Clear separation between public, semi-private, and isolated subnets## Additional Considerations

- When using 3 AZs with `zone_bits = 2`, you get 4 possible subnets per block but only use 3. The unused subnet (idx=3) is simply not referenced.
- The first-level split using `newbits = 2` gives us 4 possible blocks, but we only use 3 (netnum 0, 1, 2). The fourth block (netnum 3) is available for future expansion.
- This same pattern can be applied to secondary CIDR blocks (like EKS CIDR blocks) using the same logic.
- The approach ensures that all subnets within a tier (web/backend/datastore) have the same size, which simplifies routing and security group management.

## Key Takeaways

- Use `cidrsubnet(prefix, 2, 0/1/2)` to split a CIDR into three equal blocks (web, backend, datastore)
- Use `zone_bits = 1` for 2 AZs, `zone_bits = 2` for 3 AZs
- Iterate with `for_each` or `for` loops to create subnets per AZ
- The `idx` from your loop becomes the `netnum` parameter
- Web subnets get Internet Gateway, backend subnets get NAT Gateway, datastore subnets get neither

This approach has served me well for multiple VPC creations, and having it documented here will save me time in the future!