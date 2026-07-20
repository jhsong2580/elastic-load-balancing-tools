# Automatic Traffic Mirroring for ALB/CLB

## What is this?

A CloudFormation-based automation tool that sets up VPC Traffic Mirroring for ALB/CLB and continuously captures packet data to S3 — without manual intervention. This tool supports AWS commercial partitions only (not aws-cn or aws-us-gov).

When deployed, it:
- Creates dedicated EC2 instances (one per AZ) to receive mirrored traffic
- Automatically detects ALB/CLB ENI changes (scale-out/scale-in) via EventBridge
- Captures all mirrored packets using tcpdump and uploads pcap files to S3
- Cleans up all resources when the stack is deleted

This eliminates the need for the manual 9-step Traffic Mirroring setup process and handles ALB node replacement automatically.

## How it Works

### Stack Creation
1. **ParseELB** Lambda fetches ALB/CLB info, validates subnets, builds AZ-to-subnet mapping
2. CloudFormation creates infrastructure: S3 bucket, IAM roles, Security Group (UDP 4789 for VXLAN + TCP 22 for SSH, both from VPC CIDR), Traffic Mirror Filter
3. **Traffic Mirror Controller** Lambda creates one EC2 per AZ with UserData (vxlan1 + tcpdump + S3 polling uploader) and one Mirror Target per EC2
4. **Initial Lambda** queries existing ALB/CLB ENIs and invokes Mirror Session Creator for each
5. **Mirror Session Creator** creates a Traffic Mirror Session per ENI, reusing existing AZ targets

### ALB Scale-Out (new node added)
1. ALB creates new ENI → CloudTrail records `CreateNetworkInterface`
2. EventBridge matches ENI description pattern → triggers Mirror Session Creator
3. Mirror Session Creator creates a new session pointing to the existing AZ target
4. Mirrored traffic flows immediately to the existing EC2

### ALB Scale-In (node removed)
- ALB ENI is deleted → Mirror Session is automatically removed by AWS (source ENI gone)
- EC2 and Mirror Target remain intact for future ENIs in that AZ

### Stack Deletion
1. Traffic Mirror Controller deletes all Mirror Sessions, Mirror Targets, and terminates EC2s
2. EC2 ExecStop handler uploads remaining pcap files to S3 before shutdown
3. CloudFormation removes all other resources
4. S3 bucket is preserved (DeletionPolicy: Retain)

## Prerequisites

- **CloudFormation Deploy Role**: Create an IAM Role for CloudFormation using the files in `role/` directory:
  ```bash
  aws iam create-role \
    --role-name TrafficMirrorCfnDeployRole \
    --assume-role-policy-document file://role/deploy-role-trust-policy.json

  aws iam put-role-policy \
    --role-name TrafficMirrorCfnDeployRole \
    --policy-name DeployPolicy \
    --policy-document file://role/deploy-role-policy.json
  ```
- **S3 Access**: The Traffic Mirror Target subnet's route table must have a route to S3. This can be achieved via Internet Gateway, NAT Gateway, S3 Gateway Endpoint, or S3 Interface Endpoint in the same VPC. Gateway Endpoint or Interface Endpoint is recommended.
- **CloudTrail**: Must be enabled in the region (required for EventBridge to detect ENI creation)
  - https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-create-a-trail-using-the-console-first-time.html
- **Traffic Mirror Target subnet**: Must be in the same VPC as the ALB/CLB, one per AZ


## Setup

### Parameters

| Parameter                          | Description                                 | Example / Default                       |
|------------------------------------|---------------------------------------------|-----------------------------------------|
| ELBName                            | Name of ALB or CLB to mirror                | `load-balancer-lab`                     |
| ELBType                            | Type of load balancer                       | `ALB` or `CLB`                          |
| TrafficMirroringTargetSubnets      | Subnets for mirror target EC2s (same VPC as ALB/CLB). If fewer subnets than ALB/CLB AZs are selected, traffic from uncovered AZs is automatically routed cross-AZ. This works correctly but incurs cross-AZ data transfer charges. For cost optimization, select one subnet per ALB/CLB AZ. | Select from dropdown                    |
| TrafficMirroringTargetInstanceType | EC2 instance type for capture               | `c5.xlarge` (default)                   |
| ExistingBucketArn                  | (Optional) Existing S3 bucket ARN           | `arn:aws:s3:::my-bucket` or leave empty |
| KeyPairName                        | (Optional) EC2 Key Pair for SSH access. Leave empty to launch without a login key. Note: TCP 22 is still open from VPC CIDR regardless. | Select from dropdown or leave empty     |
| PcapRotateSeconds                  | Pcap file rotation interval in seconds      | `1800` (default, 30 min)                |
| PcapMaxSizeMB                      | Pcap file max size in MB before rotation    | `500` (default)                         |
| PcapSnapLen                        | Packet snapshot length in bytes             | `0` (default, full packet)              |

### Deploy via Console

0. Verify template integrity: `shasum -a 256 -c reuse-tmt-template.yaml.sha256`
1. Go to CloudFormation → Create Stack → Upload `reuse-tmt-template.yaml`
2. Fill in parameters
3. Under "Permissions", select `TrafficMirrorCfnDeployRole` as the IAM role
4. Check "I acknowledge that AWS CloudFormation might create IAM resources with custom names"
5. Create Stack


### Pcap File Location

Files are stored in S3 with the following path structure:
```
s3://{bucket}/{YYYY}/{MM}/{DD}/{HH}/{elb_name}_{az}_capture_{timestamp}.pcap
```

### When are pcap files uploaded to S3?

A background uploader service runs every 5 seconds and uploads completed pcap files to S3. A pcap file is considered "complete" when:
- **Time rotation (-G)**: The rotation interval has elapsed (default: 1800 seconds / 30 min), and tcpdump closes the current file and starts a new one.
- **Size rotation (-C)**: The file reaches the max size limit (default: 500 MB), and tcpdump closes it and starts a new one.
- **Stack deletion**: EC2 ExecStop handler uploads any remaining in-progress pcap files before shutdown.

### Pcap Capture Options

| Option | Parameter | Description |
|--------|-----------|-------------|
| `-G` | PcapRotateSeconds | Rotate the pcap file every N seconds. Each rotated file is immediately eligible for S3 upload. Lower values = more frequent uploads but more small files. |
| `-C` | PcapMaxSizeMB | Rotate the pcap file when it reaches N MB. Prevents excessively large files on high-traffic ALBs. |
| `-s` | PcapSnapLen | Capture only the first N bytes of each packet. Use `0` for full packet capture. Set to e.g. `96` to capture headers only (reduces file size significantly). |
