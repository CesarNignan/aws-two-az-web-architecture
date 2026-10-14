# Security Design

## Overview

This project uses AWS security groups and network segmentation to control traffic between the Application Load Balancer (ALB), bastion host, and private Amazon Elastic Compute Cloud (EC2) instances.

The security design follows the principle of **least privilege**: each component should accept only the network traffic required for its role.

The web servers run in private subnets, while the ALB and bastion host are deployed in public subnets.

---

## 1. Security Groups

Each component has its own security group to control inbound and outbound traffic.

### ALB Security Group

The ALB security group allows inbound HTTP traffic on TCP port 80.

| Direction | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| Inbound | HTTP | 80 | `0.0.0.0/0` | Allow public access to the web application |

The ALB receives incoming web requests and forwards them to healthy EC2 instances registered with its target group.

**Security note:** The application currently uses HTTP. HTTPS has not been configured because no certificate or HTTPS listener has been created.

### Bastion Host Security Group

The bastion host provides an administrative entry point for connecting to EC2 instances in private subnets.

| Direction | Protocol | Port | Source/Destination | Purpose |
|---|---|---|---|---|
| Inbound | SSH | 22 | Administrator's public IP address | Allow secure administrative access |
| Outbound | SSH | 22 | Private EC2 security group | Allow SSH connections to private instances |

The bastion host uses an SSH key pair for authentication.

**Security consideration:** SSH access should be restricted to the administrator's public IP address rather than allowing connections from anywhere. A private key provides authentication, but restricting the source IP reduces the number of systems that can attempt to connect.

### Private EC2 Security Group

The private EC2 instances use a dedicated security group to prevent direct public access and restrict communication to trusted components.

| Direction | Protocol | Port | Source/Destination | Purpose |
|---|---|---|---|---|
| Inbound | HTTP | 80 | ALB security group | Allow application traffic from the load balancer |
| Inbound | SSH | 22 | Bastion host security group | Allow administrative access through the bastion host |

The HTTP rule references the ALB security group instead of allowing traffic from any IP address. This means that web traffic is permitted only from network interfaces associated with the ALB security group, subject to the applicable routing and security rules.

The SSH rule references the bastion host security group, preventing other sources from connecting through this rule.

The private instances do not need public inbound HTTP or SSH rules.

---

## 2. Network Segmentation

The architecture separates public-facing components from application servers.

### Public Subnets

The public subnets contain:

- The internet-facing Application Load Balancer.
- The bastion host.

These resources provide public application access and a controlled administrative entry point.

### Private Subnets

The private subnets contain the EC2 web servers.

The instances do not have direct public inbound access. Application requests reach them through the ALB, while administrative SSH connections pass through the bastion host.

This separation reduces the public exposure of the application servers.

---

## 3. Application Traffic Flow

The intended application traffic path is:

**Internet → ALB → Private EC2 instances**

1. A user sends an HTTP request to the ALB's public DNS name.
2. The ALB accepts the request through its security group on TCP port 80.
3. The ALB forwards the request to a healthy target in the target group.
4. The private EC2 instance receives the request because its security group permits HTTP traffic from the ALB security group.
5. The instance returns the HTTP response through the established network connection.

The private instances do not accept direct public HTTP connections.

The ALB performs health checks on its registered targets to determine which instances are healthy enough to receive application traffic.

---

## 4. Administrative Access Flow

The intended administrative access path is:

**Administrator's computer → Bastion host → Private EC2 instance**

1. The administrator connects to the bastion host using SSH.
2. The SSH connection is authenticated using the appropriate private key.
3. From the bastion host, the administrator initiates an SSH connection to a private EC2 instance.
4. The private instance accepts the connection because its security group permits SSH traffic from the bastion host security group.

This arrangement allows administrative access without exposing the private instances directly to the internet.

**Security note:** Private keys must be stored securely and must never be committed to the GitHub repository.

---

## 5. Outbound Internet Access

The private subnets use NAT Gateways to provide outbound internet connectivity.

A Network Address Translation (NAT) Gateway allows resources in private subnets to initiate connections to external services without allowing unsolicited inbound internet connections to those resources.

In this architecture, the NAT Gateways support outbound connectivity from the private instances when required.

The NAT Gateways are not involved in communication between the ALB and the private EC2 instances. That traffic uses the VPC's internal routing.

---

## 6. Security Principles Applied

### Least Privilege

Each security group allows only the traffic required for its component's role.

### Network Segmentation

Public-facing resources and private application servers are placed in separate subnets.

### Restricted Administrative Access

SSH access to private instances is allowed only from the bastion host security group.

### Controlled Application Access

HTTP access to private instances is allowed only from the ALB security group.

### Reduced Public Exposure

Private EC2 instances are not intended to be accessed directly from the public internet.

### Credential Protection

SSH private keys must remain outside the repository and must be stored securely.

---

## 7. Security Limitations and Future Improvements

The current implementation provides basic network isolation and controlled access between the architecture's components. Further improvements could include:

- **Configure HTTPS:** Obtain and configure a valid TLS certificate and an HTTPS listener on the ALB to encrypt traffic between users and the load balancer.
- **Restrict bastion access:** Verify that inbound SSH is limited to the administrator's trusted public IP address.
- **Consider AWS Systems Manager Session Manager:** Evaluate replacing the publicly reachable bastion host with managed administrative access.
- **Review security group rules:** Periodically remove unnecessary rules and verify that each rule has an appropriate source and purpose.
- **Improve credential management:** Avoid storing private keys or other secrets in source control.
- **Enable monitoring and auditing:** Use appropriate AWS logging and monitoring services to investigate access and security events.

---

## 8. Verification Checklist

Before considering the security configuration complete, verify the following in the AWS Management Console:

- [ ] The ALB security group allows inbound HTTP on TCP port 80.
- [ ] No HTTPS listener is documented as active unless one has actually been configured.
- [ ] The bastion security group allows SSH only from the administrator's trusted public IP address.
- [ ] The private EC2 security group allows HTTP only from the ALB security group.
- [ ] The private EC2 security group allows SSH only from the bastion host security group.
- [ ] The private EC2 instances do not have unnecessary public inbound rules.
- [ ] No private keys, credentials, or other secrets are present in the repository.

---

## Conclusion

The architecture uses security groups and subnet separation to control how users, administrators, and AWS resources communicate.

The ALB is the public entry point for application traffic, the bastion host provides an administrative access path, and the private EC2 instances accept HTTP traffic from the ALB and SSH traffic from the bastion host.

This design reduces direct exposure of the web servers and establishes a foundation for further security improvements.