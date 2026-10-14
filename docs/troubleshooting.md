## 2. Issue: Private Instances Could Not Reach the Internet

### Symptom

During testing, I attempted to use `ping` from a private EC2 instance to check internet connectivity, but it did not work.

Rather than immediately changing the configuration, I investigated the network setup to identify the cause.

### Investigation

I reviewed the network configuration, including the placement of the NAT Gateway and the routing used by the private subnet.

I discovered that the NAT Gateway had been deployed in a private subnet instead of a public subnet.

This prevented the private instances from using the intended NAT Gateway path to reach the internet.

### Root Cause

The NAT Gateway was incorrectly placed in a private subnet.

A public NAT Gateway must be deployed in a public subnet whose route table provides a route to an Internet Gateway. The private subnet's route table must then direct internet-bound traffic to the NAT Gateway.

### Resolution

I corrected the NAT Gateway placement so that it could use the intended public internet path.

After correcting the configuration, I repeated my connectivity test and confirmed that it worked.

### Expected Network Flow

Private EC2 instance
    ↓
Private subnet route table
    ↓
NAT Gateway in a public subnet
    ↓
Public subnet route table
    ↓
Internet Gateway
    ↓
Internet

### Verification

My initial test used `ping`. However, ping relies on Internet Control Message Protocol (ICMP), and some destinations block this traffic.
