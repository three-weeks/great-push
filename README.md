# ICP Deployment on AWS

The material here is intended to track and harvest the deployment experience and assets of deploying ICP 2.1.0.2 on AWS cloud platform.

* [Infrastructure Architecture](#infrastructure-architecture)
* [Architectural Decisions and Questions](#architectural-decisions)
* [Terraform Automation](#terraform-automation)
* [Installation Procedure](#installation-procedure)
* [Cluster access](#cluster-access)
* [AWS Cloud Provider](#aws-cloud-provider)
* [Master Node Recovery](#master-node-recovery)

## Infrastructure Architecture
The Terraform variables can be modified to deploy either a singe master configuration or a multiple master architecture. 

The following diagram outlines the infrastructure architecture for the multiple master deployment.

  ![Infrastructure Architecture](imgs/icp_ha_aw_overview.png?raw=true)

In a single master, we divide the network into a public subnet which is directly connected to the internet, and a private subnet that can reach the internet through the NAT gateway:



Fix this diagram.



  ![Single Availability Zone Infrastructure](imgs/icp_ha_aws_single_az.png?raw=true)



## Terraform Automation

### Prerequisites

1. To use Terraform automation, download the Terraform binaries [here](https://www.terraform.io/).
2. Create an S3 bucket in the same region that the ICP cluster will be created and upload the ICP binaries.  Make note of the bucket name.  You can use the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/awscli-install-bundle.html) to do this.
3. Create an AMI and install the AWS CLI. Make sure the AMI meets the following requirements.
  1. <<<< Add sizing and other AMI reqs. here>>>>
4. Convert the instance into an AMI.
5. Create a file, `terraform.tfvars`, in the `terraform` directory, containing the values for the following:

|name | required                        | value        |
|----------------|------------|--------------|
| access_key   | yes          | AWS access key id                           |
| secret_key   | yes          | AWS access key secret                       |
| aws_region   | no           | AWS region that the VPC will be created in.  By default, uses `us-east-2`.  Note that the AWS selected region should have at least 3 availability zones. |
| key_name | yes | AWS keypair name to assign to instances |
| Internal_elb | no | If true, the ELB will be internal. Default is "true" |
| existing_vpc_id | Yes | The ID of the existing VPC to deploy the code into. |
| azs | no | An array of the AWS availability zones that will be appended to the region variable.  Defaults to (a, b, c). |
| existing_subnet_id | Yes | An array of the subnets to deploy the infrastructure into. Should corespond with the values of the "azs" value. |
| existing_default_security_group_id | Yes | The ID of the default security group for the VPC. |
| existing_kms_key_id | Yes | The ID of the AWS KMS key to be used during EBS disc encryption. |
| allowed_cidr_master_8001 | Yes | An array of the ingress CIDRs that are allowed to communicate with the master nodes on port 8001. |
| allow_cidr_master_8443 | Yes | An array of the ingress CIDRs that are allowed to communicate with the master nodes on port 8443. |
| allow_cidr_master_8500 | Yes | An array of the ingress CIDRs that are allowed to communicate with the master nodes on port 8500. |
| allow_cidr_proxy_80 | Yes | An array of the ingress CIDRs that are allowed to communicate with the proxy nodes on port 80. |
| allow_cidr_proxy_443 | Yes | An array of the ingress CIDRs that are allowed to communicate with the proxy nodes on port 443. |

See [Terraform documentation](https://www.terraform.io/intro/getting-started/variables.html) for the format of this file.

Alternatively, these can be specified in environment variables with the `TF_VAR_` prefix.  e.g.

```bash
export TF_VAR_key_name=<my keypair name>
```

6. Initialize Terraform from the `terraform` directory using this command.  This will download all dependent modules, including the [ICP installation module](https://github.com/hassenius/terraform-module-icp-deploy). (Requires internet access)

   ```bash
   terraform init
   ```

### Run the Terraform Automation

Run this command to see what would be created in the AWS account:

```bash
terraform plan
```

To move forward and create the objects, use the following command:

```bash
terraform apply
```

When the installation completes,  the `/opt/ibm/cluster` directory on the boot master (i.e. `icp-master1`) is backed up to S3 in a bucket named `icpbackup-<clusterid>`, which can be used in master recovery in case one of the master nodes fails.  It is recommended after every time `terraform apply` is performed, to commit the `terraform.tfstate` into git so that the state is stored in source control.

When installation completes, if a user-provided certificate is used, create a CNAME entry in your DNS provider from the DNS entry to the ELB DNS URL output at the end of the terraform process.

### Terraform objects

The Terraform automation creates the following objects.

#### EC2 Instances

*these are tagged with the cluster id for Kubernetes-AWS integration*

| Node Role | Count | AWS EC2 Instance Type | Subnet | Security Group(s) |
|-----------|-------|-----------------------|--------|-------------------|
| Bastion   |   2   | t2.large              | public | icp-default, icp-bastion |
| Master    |   3   |  m4.xlarge            | private | icp-default, icp-master |
| Management |  3   | m4.xlarge             | private | icp-default, icp-management |
| Proxy     |   3   | m4.large              | private | icp-default, icp-proxy |
| Worker    |  > 3   | m4.xlarge             | private | icp-default, icp-worker |

*(the instance types, base AMIs and counts can be configured in `variables.tf`)*

#### Elastic Network Interfaces

For recovery, master and proxy nodes have Network Interfaces created and attached to them as the first network device.  The private IP address is bound to the network interface, so when the interface is attached to a newly created instance, the IP address is preserved.  This is useful for Master Recovery, which is covered in below.

#### IAM Configuration

An IAM role is created in AWS and attached to each EC2 instance with the following policy:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:Describe*",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "ec2:AttachVolume",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "ec2:DetachVolume",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["ec2:*"],
      "Resource": ["*"]
    },
    {
      "Effect": "Allow",
      "Action": ["elasticloadbalancing:*"],
      "Resource": ["*"]
    }
  ]
}
```

*(The Kubernetes AWS Cloud Provider needs access to read information from the AWS API about the instance (i.e. which subnet it's in, the private DNS name, whether nodes that have been removed, etc), and to create LoadBalancers and EBS Volumes on demand.)*

Additionally, we add `S3FullAccess` policy so that the IAM role can get installation images out of an S3 bucket and back up the `/opt/ibm/cluster` directory to an S3 bucket after installation.

#### VPC
- VPC with an internet gateway   
- All ICP nodes are in private subnet, each with own NAT Gateway
  - outbound Internet access from private subnet through NAT Gateway

#### Subnets
- two for each AZ (one public, one private)
  - *(CIDR for VPC and subnets can be configured in `variables.tf`)*
  - *(these are tagged with the cluster id for Kubernetes ELB integration)*

#### Security Group
- `icp-bastion`
  - allow 22 from 0.0.0.0/0   
- `icp-default`
  - allow ALL traffic from itself *(all nodes are in this security group)*
  - *(this is tagged with the cluster id for Kubernetes ELB integration)*
- `icp-proxy`
  - allow from 0.0.0.0/0 on port 80
  - allow from 0.0.0.0/0 on port 443
  - *(note that Network Load Balancers do not allow security groups)*
- `icp-master`
  - allow from 0.0.0.0/0 on port 9443 (auth service)   
  - allow from 0.0.0.0/0 on port 8500 (image registry)   
  - allow from 0.0.0.0/0 on port 8443 (master UI)   
  - allow from 0.0.0.0/0 on port 8001 (kube api)   
- `icp-workers`
  - allow all traffic from self (future use)
- `icp-management`
  - allow all traffic from self (future use)
- efs mounts
  - allow all from icp master nodes

#### EFS Storage
- two volumes (both at least 20GB)   
  - /var/lib/registry (image registry)
  - /var/lib/icp/audit (audit logs)

#### Load Balancer
- Network LoadBalancer for ICP console
  - listen on 8443, forward to master nodes port 8443 (ICP Console)
  - listen on 8001, forward to master nodes port 8001 (Kubernetes API)
  - listen on 8500, forward to master nodes port 8500 (Image registry)
  - listen on 9443, forward to master nodes port 9443 (Auth service)
- Network LoadBalancer for ICP Ingress resources
  - listen on port 80, forward to proxy nodes port 80 (http)
  - listen on port 443, forward to proxy nodes port 443 (https)

#### Route53 DNS Zone

For convenience, a private DNS Zone is created in Route 53.  The domain name can be configured in `variables.tf`; by default it is `icpcluster.icp`.  The domain search suffixes are added to `resolv.conf`, but due to a bug in cloud-init, `resolv.conf` is overwritten by NetworkManager in RHEL. It should be resolved in a future release of cloud-init.

## Installation Procedure

The installer automates the install procedure described [here](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_2.1.0/installing/installing.html).

### ICP Installation Parameters on AWS

Suggested ICP Installation parameters specific on AWS:

```
---
calico_tunnel_mtu: 8981
cloud_provider: aws
kubelet_nodename: nodename
cluster_CA_domain: icp-console-b75cb66c49be7a49.elb.us-east-2.amazonaws.com
cluster_lb_address: icp-console-b75cb66c49be7a49.elb.us-east-2.amazonaws.com
proxy_lb_address: icp-proxy-e2575b7221d771dd.elb.us-east-2.amazonaws.com
```

Because AWS enables Jumbo frames (MTU 9001), the Calico IP-in-IP tunnel is configured to take advantage of the larger MTU.

The `cloud_provider` parameter allows Kubernetes to take advantage of some Kubernetes-AWS integration with dynamic ELB creation and dynamic EBS creation for persistent volumes.  When the AWS `cloud_provider` is used, all node names use the private FQDN retrieved from from the AWS metadata service and nodes are tagged with the correct region and availability zone.  Kubernetes will stripe deployments across availability zones.  See the [below section](#aws-cloud-provider) for more details.

The `cluster_CA_domain`, `cluster_lb_address`, and `proxy_lb_address` correspond to the DNS names for the master and proxy ELB.

Note the other parameters in the `icp-deploy.tf` module.  The config files are stored in `/opt/ibm/cluster/config.yaml` on the boot-master.

## Cluster access

### ICP Console access

The ICP console can be accessed at `https://<cluster_lb_address>:8443`.  See [documentation](https://www.ibm.com/support/knowledgecenter/SSBS6K_2.1.0/manage_cluster/cfc_gui.html).

### ICP Private Image Registry access

The registry is available at `https://<cluster_lb_address>:8500`.  See [documentation](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_2.1.0/manage_images/configuring_docker_cli.html) for how to configure Docker to access the registry.

### ICP Kubernetes API access

The Kubernetes API can be reached at `https://<cluster_lb_address>:8001`.  To obtain a token, see the [documentation](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_2.1.0/manage_cluster/cfc_cli.html) or this [blog post](https://www.ibm.com/developerworks/community/blogs/fe25b4ef-ea6a-4d86-a629-6f87ccf4649e/entry/Configuring_the_Kubernetes_CLI_by_using_service_account_tokens1?lang=en),

### ICP Ingress Controller

[Ingress Resources](https://kubernetes.io/docs/concepts/services-networking/ingress/) can be created and exposed using the proxy node endpoints at `http://<proxy_lb_address>:80` or `https://<proxy_lb_address>:443`

## AWS Cloud Provider

The AWS Cloud provider provides Kubernetes integration with Elastic Load Balancer and Elastic Block Store.  See documentation on [LoadBalancer](https://kubernetes.io/docs/concepts/cluster-administration/cloud-providers/#aws) and [Volume](https://kubernetes.io/docs/concepts/storage/storage-classes/#aws)

## Master Node Recovery

See [additional documentation](./MasterNodeRecovery.md).
# great-push
