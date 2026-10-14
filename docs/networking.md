# Networking Design

## Overview

The network infrastructure is built around a custom Amazon Virtual Private Cloud (VPC) with the IPv4 Classless Inter-Domain Routing (CIDR) block `10.0.0.0/24`.

The VPC spans two Availability Zones (AZs), with a public subnet and a private subnet in each zone. An Internet Gateway (IGW) provides internet connectivity for appropriately configured public resources, while two NAT Gateways allow resources in the private subnets to initiate outbound internet connections.

Route tables control how traffic moves between subnets and external networks, while security groups control which inbound and outbound connections are permitted.

This design separates public-facing infrastructure from private application servers and provides network components across two Availability Zones.

## VPC and CIDR Allocation

The VPC uses the IPv4 CIDR block `10.0.0.0/24`, which contains 256 IPv4 addresses in total.

The address space is divided into four `/26` subnets. Each subnet contains 64 addresses in its CIDR range, of which AWS reserves five IPv4 addresses.

| Subnet | CIDR | Availability Zone | Purpose |
|---|---|---|---|
| Public AZ1 | `10.0.0.0/26` | AZ1 | Bastion host, NAT Gateway, and public-facing infrastructure |
| Private AZ1 | `10.0.0.64/26` | AZ1 | Private web servers |
| Public AZ2 | `10.0.0.128/26` | AZ2 | NAT Gateway and public-facing infrastructure |
| Private AZ2 | `10.0.0.192/26` | AZ2 | Private web servers |

This subnet allocation provides separate address ranges for public and private resources in each Availability Zone.

Using distinct subnet CIDR blocks also prevents address-range overlap within the VPC.

## Subnet Design

The VPC contains two public subnets and two private subnets distributed across two Availability Zones.

### Public Subnets

The public subnets host resources that require public-facing connectivity or provide network services for other resources.

These include:

- The bastion host in Availability Zone 1.
- A NAT Gateway in each Availability Zone.
- The internet-facing Application Load Balancer, which uses both public subnets.

The public subnets use route tables that direct internet-bound traffic through the Internet Gateway.

A resource does not become publicly accessible merely because it resides in a public subnet. Its addressing, routing, and security configuration must also permit the intended connectivity.

### Private Subnets

The private subnets host the EC2 web servers running NGINX.

These instances receive application traffic through the Application Load Balancer rather than accepting direct inbound web traffic from the public internet.

Each private subnet has its own route table configured to direct internet-bound traffic through the NAT Gateway in the corresponding Availability Zone.

This separation reduces the direct exposure of application servers and allows network traffic to be controlled through routing rules and security groups.

## Internet Gateway

An Internet Gateway (IGW) is attached to the VPC and provides internet connectivity for resources with the appropriate network configuration.

The public route tables contain a default route:

`0.0.0.0/0 → Internet Gateway`

This route directs traffic destined for addresses outside the VPC's local network through the Internet Gateway.

The internet-facing Application Load Balancer uses this public network configuration to receive HTTP requests from users on the internet.

The bastion host can also be accessed remotely when its addressing, routing, and security-group rules permit the connection.

An Internet Gateway does not automatically make every resource in the VPC publicly accessible. Public addressing, appropriate routing, and security controls are also required.

## Route Tables and Routing

Route tables determine where network traffic is directed based on its destination.

The architecture uses public and private routing configurations to separate internet-facing resources from private application servers.

### Public Route Tables

The public subnets use route tables containing a default route pointing to the Internet Gateway.

Conceptually, the route is:

| Destination | Target |
|---|---|
| `10.0.0.0/24` | Local |
| `0.0.0.0/0` | Internet Gateway |

The local route allows communication between resources using addresses within the VPC's address space, subject to applicable network security controls.

The default route directs traffic destined for other IPv4 networks through the Internet Gateway.

### Private Route Tables

The private subnets use route tables that direct internet-bound traffic through their respective NAT Gateways.

Conceptually, the routing configuration is:

| Private subnet | Destination | Target |
|---|---|---|
| Private AZ1 | `10.0.0.0/24` | Local |
| Private AZ1 | `0.0.0.0/0` | NAT Gateway AZ1 |
| Private AZ2 | `10.0.0.0/24` | Local |
| Private AZ2 | `0.0.0.0/0` | NAT Gateway AZ2 |

The corresponding private route table must be associated with each private subnet for these routes to apply.

This configuration allows private instances to initiate outbound internet connections without requiring direct inbound internet connectivity.

The local route also allows the Application Load Balancer to communicate with private web servers through the VPC's internal network, provided the relevant security controls permit the traffic.

## NAT Gateway Design

The architecture uses two NAT Gateways, with one deployed in the public subnet of each Availability Zone.

A NAT Gateway allows resources in private subnets to initiate connections to destinations outside the VPC without requiring those private instances to have public IPv4 addresses.

For example, a private EC2 instance may need outbound connectivity to:

- Download operating system updates.
- Install software packages.
- Reach external services required by an application.

The private subnet in Availability Zone 1 routes internet-bound traffic through NAT Gateway AZ1, while the private subnet in Availability Zone 2 routes internet-bound traffic through NAT Gateway AZ2.

This arrangement reduces dependence on a NAT Gateway located in the other Availability Zone and can reduce cross-AZ data-transfer costs.

NAT Gateways do not independently provide unsolicited inbound internet access to private instances.

They are also not required for communication between the Application Load Balancer and web servers inside the VPC.

NAT Gateways can incur hourly and data-processing charges, so their cost should be considered when operating the lab.

## Traffic Flows

### Inbound Application Traffic

When a user accesses the application through the Application Load Balancer's DNS name, the request follows this general path:

1. The user's browser sends an HTTP request to the Application Load Balancer.
2. The Internet Gateway provides the internet connectivity required by the internet-facing load balancer.
3. The load balancer evaluates its listener rules and forwards the request to a healthy registered target.
4. The selected private EC2 instance receives the request on HTTP port 80.
5. NGINX processes the request and returns an HTTP response.
6. The Application Load Balancer returns the response to the user.

The Application Load Balancer is configured to use both public subnets and forwards application traffic to web servers in the private subnets.

The security groups restrict HTTP access to the web servers so that the load balancer's security group is the permitted source.

If a target becomes unhealthy, the load balancer can stop directing new requests to that target and use other healthy registered targets.

Repeated browser refreshes do not guarantee that requests will alternate between Availability Zones. Traffic distribution depends on the load balancer's configuration and the health and availability of its targets.

### Administrative Access

Administrative access to private EC2 instances follows a bastion-based Secure Shell (SSH) connection path.

The general connection flow is:

`Administrator → Bastion Host → Private EC2 Instance`

The administrator first connects to the bastion host in the public subnet of Availability Zone 1.

From the bastion host, the administrator can initiate an SSH connection to a private EC2 instance, provided the applicable network and security-group rules allow the connection.

The bastion host's security group restricts inbound SSH access to the administrator's public IP address.

The private EC2 instances' security group permits SSH connections from the bastion host's security group rather than from arbitrary internet addresses.

The private instances do not need public IP addresses to receive these administrative connections through the bastion host.

This approach reduces direct exposure of the private servers to the public internet.

Because the bastion host is located in Availability Zone 1, administrative access through this host depends on the availability and connectivity of that host. A production design could consider alternatives such as AWS Systems Manager Session Manager to reduce dependence on a publicly reachable bastion host.

### Outbound Internet Access

Private instances can initiate outbound internet connections through their respective NAT Gateways when their route tables direct internet-bound traffic to those gateways.

For example, an EC2 instance in Private AZ1 sends a request to an external software repository.

The traffic follows this general path:

`Private EC2 AZ1 → Private Route Table AZ1 → NAT Gateway AZ1 → Internet Gateway → Internet`

The response returns through the established network connection.

The equivalent path applies to private instances in Availability Zone 2, using the NAT Gateway and route table associated with that zone.

This arrangement supports outbound connectivity without requiring the private instances to be directly reachable from the public internet.

The NAT Gateway must be deployed in a public subnet with a suitable route to the Internet Gateway, and the private route table must direct internet-bound traffic to the correct NAT Gateway.

Outbound connectivity also depends on security-group rules, network access control lists, and the availability of the destination service.

Communication between the Application Load Balancer and the private web servers follows the VPC's internal network path. It does not require a NAT Gateway.

## Network Security

The architecture uses security groups to control which resources can communicate with one another.

The principal security-group relationships are:

| Security group | Traffic | Permitted source or destination |
|---|---|---|
| `ALB-SG` | Inbound HTTP, port 80 | Internet |
| `ALB-SG` | Outbound HTTP, port 80 | `Private-EC2-SG` |
| `Private-EC2-SG` | Inbound HTTP, port 80 | `ALB-SG` |
| `Private-EC2-SG` | Inbound SSH, port 22 | `Bastion-SG` |
| `Bastion-SG` | Inbound SSH, port 22 | Administrator's public IP address |
| `Bastion-SG` | Outbound SSH, port 22 | `Private-EC2-SG` |

These entries describe the intended security-group configuration and should be checked against the actual rules in the AWS Console.

Security-group references are useful because they allow traffic from resources associated with a particular security group without requiring individual instance IP addresses to be specified.

For example, allowing inbound HTTP traffic from `ALB-SG` on the private web servers restricts application access to traffic originating from network interfaces associated with that security group.

Similarly, allowing inbound SSH traffic from `Bastion-SG` restricts administrative access to connections originating from resources associated with the bastion security group.

The architecture avoids opening SSH access to `0.0.0.0/0`, which would permit connection attempts from any IPv4 address at the security-group level.

Security groups are stateful, meaning response traffic for permitted connections is automatically allowed, subject to the applicable security-group behavior.

Network access control lists, route tables, and instance-level configuration can also affect connectivity and should be considered when troubleshooting network problems.

## Network Validation

The network should be validated by checking both its configuration and its actual behavior.

The following checks help confirm that the intended network design is implemented correctly.

### VPC and Subnets

- Confirm that the VPC uses `10.0.0.0/24`.
- Verify that all four subnets use the intended CIDR blocks.
- Confirm that the subnets are distributed across two distinct Availability Zones.
- Check that the Application Load Balancer uses both public subnets.
- Verify that the web servers are deployed in the intended private subnets.

### Route Tables

- Confirm that the public route tables direct internet-bound IPv4 traffic to the Internet Gateway.
- Verify that each private subnet is associated with the correct private route table.
- Confirm that each private route table directs internet-bound traffic to its corresponding NAT Gateway.
- Verify that the VPC's local route is present.

### Internet and NAT Connectivity

- Confirm that the Internet Gateway is attached to the VPC.
- Verify that each NAT Gateway is available and deployed in a public subnet.
- Confirm that each NAT Gateway has a route to the Internet Gateway through its public subnet's route table.
- Check whether private instances can reach the external services they require.

### Security Groups

- Verify that the Application Load Balancer permits inbound HTTP traffic on port 80.
- Confirm that the load balancer can communicate with the web servers on the intended application port.
- Verify that the private web servers accept HTTP traffic from `ALB-SG`.
- Confirm that SSH access to the bastion host is restricted to the administrator's public IP address.
- Verify that SSH access to private instances is permitted from `Bastion-SG`, rather than arbitrary internet addresses.

### Application Connectivity

- Confirm that both web servers are running NGINX.
- Verify that each web server serves its expected test page.
- Confirm that both targets are registered in the Application Load Balancer's target group.
- Check that both targets become healthy when operating normally.
- Access the application through the load balancer's DNS name.
- Test whether requests continue to succeed when one web server becomes unavailable.

During testing, I observed that the Application Load Balancer continued directing requests to healthy targets when a web server became unavailable.

This demonstrated traffic failover between healthy targets in the two Availability Zones.

Further testing should confirm that Auto Scaling replaces failed instances, that replacement instances become healthy targets, and that the architecture behaves as expected during more extensive failures.

These checks help distinguish between a network configuration that appears correct in the console and a network that has been verified through actual connectivity tests.