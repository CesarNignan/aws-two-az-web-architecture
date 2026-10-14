# Architecture Details

## Architecture Goals

The primary goal of this project was to build a web application architecture across two AWS Availability Zones (AZs) to improve availability and reduce dependence on a single server or location.

The project helped me understand how networking, compute, load balancing, security, and automatic instance replacement work together in a cloud environment.

The architecture was built manually through the AWS Management Console as a hands-on Cloud Engineering project.

## High-Level Design

The architecture uses a custom Amazon Virtual Private Cloud (VPC) with the CIDR block `10.0.0.0/24`, divided into four subnets across two Availability Zones.

Each Availability Zone contains:
- One public subnet.
- One private subnet.
- A NAT Gateway in the public subnet.
- A private Amazon Elastic Compute Cloud (EC2) web server managed by an Auto Scaling Group (ASG).

The first Availability Zone also contains a bastion host for administrative access to private instances.

An internet-facing Application Load Balancer (ALB) spans both public subnets and distributes incoming HTTP requests to registered web servers in the private subnets.

An Internet Gateway (IGW) provides the VPC's internet connectivity, while route tables determine how traffic reaches the appropriate destinations.

## Availability Zone 1

Availability Zone 1 contains a public subnet and a private subnet.

The public subnet hosts:
- **Bastion host:** Provides an administrative entry point for securely accessing private EC2 instances over Secure Shell (SSH).
- **NAT Gateway:** Allows resources in the private subnet to initiate outbound internet connections when their route tables direct traffic through it.

The private subnet hosts an EC2 web server running NGINX and an Auto Scaling Group responsible for maintaining the configured instance capacity.

The private server's security group allows inbound SSH traffic from the bastion host's security group. HTTP traffic from the Application Load Balancer is also permitted.

The private instance does not need to accept direct inbound web traffic from the public internet.

## Availability Zone 2

Availability Zone 2 follows a similar design, with a public subnet and a private subnet.

The public subnet contains a NAT Gateway, while the private subnet contains an EC2 web server running NGINX and an associated Auto Scaling Group.

Unlike Availability Zone 1, Availability Zone 2 does not contain a separate bastion host.

Administrative access to private instances in this Availability Zone uses the bastion host in Availability Zone 1, subject to the applicable security-group rules and network connectivity.

The web server is registered with the same Application Load Balancer target group as the server in Availability Zone 1.

This design allows the load balancer to distribute requests across web servers in both Availability Zones.

## Request Flow

When a user accesses the application through the Application Load Balancer's DNS name, the request follows this general path:

1. The user's browser sends an HTTP request to the Application Load Balancer.
2. The Internet Gateway provides the internet connectivity required for the internet-facing load balancer.
3. The load balancer evaluates its listener rules and forwards the request to a healthy registered target.
4. The selected private EC2 instance receives the request on port 80.
5. NGINX processes the request and returns an HTTP response.
6. The Application Load Balancer returns the response to the user.

The load balancer distributes requests according to its configuration and target availability. Repeated browser refreshes do not guarantee that requests will alternate between Availability Zones.

## Administrative Access Flow

Administrative access to private EC2 instances follows a bastion-based SSH connection path:

1. The administrator connects to the bastion host using SSH.
2. The bastion host acts as the entry point into the private network.
3. From the bastion host, the administrator connects to the required private EC2 instance using SSH.

The bastion host's security group restricts inbound SSH access to the administrator's public IP address.

The private EC2 instances' security group permits SSH connections from the bastion host's security group rather than from arbitrary internet addresses.

This approach reduces direct exposure of the private servers to the public internet.

## Outbound Internet Access

Private instances can use NAT Gateways to initiate outbound internet connections, provided the relevant private route tables direct internet-bound traffic to the appropriate NAT Gateway.

For example, a private instance may need outbound connectivity to download operating system updates or install software packages.

A NAT Gateway does not automatically provide inbound internet access to private instances.

The Application Load Balancer communicates with registered targets through the VPC's internal network path. This communication does not require a NAT Gateway.

## Auto Scaling Design

The architecture uses separate Auto Scaling Groups for the web servers in the two Availability Zones.

Each group uses a launch template to define the configuration of instances it launches.

The launch templates specify the required instance configuration, while the Auto Scaling Groups determine the desired capacity and subnet placement.

During testing, I experimented with different capacity settings, including increasing the maximum capacity to six instances.

The capacity settings determine how many instances a group is permitted to maintain, but they do not by themselves guarantee high availability.

For reliable recovery, the launch templates must correctly recreate the required server configuration, and the load balancer must be able to register and route traffic to healthy replacement instances.

**Important limitation:** Because the Auto Scaling Groups are associated with separate Availability Zones, the architecture does not automatically guarantee that one group will increase its capacity when the other Availability Zone fails. Cross-Availability-Zone recovery behavior requires additional configuration and testing.

## Health Checks and Failure Handling

The Application Load Balancer uses target health checks to determine whether registered web servers are healthy enough to receive traffic.

The Auto Scaling Groups also monitor instance health and maintain their configured capacity by replacing instances when the applicable health checks and replacement policies indicate that replacement is necessary.

These are related but distinct mechanisms:

- **Load balancer health checks:** Determine whether a target should receive application traffic.
- **Auto Scaling health monitoring:** Helps determine whether an instance needs to be replaced.

The health-check interval, thresholds, and replacement behavior depend on the actual configuration.

A replacement instance must also start successfully, run NGINX, and pass the required health checks before it can reliably serve application traffic.

## Limitations and Improvements

This project provided practical experience with AWS networking, private compute, load balancing, Auto Scaling, and security-group configuration.

Several improvements would make the architecture more robust and reproducible:

- **Infrastructure as Code:** Use Terraform to provision and manage infrastructure through version-controlled configuration.
- **Repeatable server configuration:** Use launch-template user data or a preconfigured Amazon Machine Image (AMI) to ensure replacement instances consistently run the intended application.
- **Availability testing:** Simulate an instance failure and verify that traffic continues to reach healthy targets.
- **Auto Scaling improvements:** Evaluate a single Auto Scaling Group spanning multiple Availability Zones and test how it responds when capacity is lost in one zone.
- **Monitoring and alerting:** Add monitoring for instance health, target health, resource utilization, and application failures.
- **Security improvements:** Review administrative access, permissions, and opportunities to reduce reliance on a publicly reachable bastion host.
- **Cost optimization:** Review NAT Gateway, load balancer, EC2, and storage costs, and remove resources when they are no longer required.
- **Deployment automation:** Develop a repeatable deployment process for the web application and its configuration.

This is a learning project rather than a production-certified architecture. Its resilience and recovery capabilities should be validated through testing before making stronger high-availability claims.