# AWS-CloudFormationTemplates
AWS CloudFormation Templates


## CloudFormation templates for a Mesosphere cluster.

Prerequisites:

An Exhibitor endpoint for ZooKeeper node discovery. c
A ZooKeeper client security group to associate with the Mesosphere nodes.

##Overview

This project includes three templates:

Mesosphere-master.json - Launch a set of Mesosphere Masters running Marathon in an auto scaling group
Mesosphere-slave.json - Launch a set of Mesosphere Slaves in an auto scaling group
Mesosphere.json - Creates both a Mesosphere-master and Mesosphere-slave stack from the corresponding templates.
In general, you'll want to launch the Mesosphere cluster via Mesosphere.json.

The servers are launched from public AMIs running Ubuntu 14.04 LTS and pre-loaded with Docker, Runit, and Mesosphere. See packer/ubuntu-14.04-Mesosphere.json for build details. If you wish to use your own AMI, simply pass MasterInstanceAmi and/or SlaveInstanceAmi to Mesosphere.json.

Marathon is run on the masters via a Docker image specified as a Parameter. 

To adjust cluster capacity, simply increment or decrement SlaveInstanceCount. You can do the same for the masters with MasterInstanceCount, but update MasterQuorumCount accordingly. CloudFormation will update the auto scaling groups, and the node addition/removal should be handled transparently by Mesosphere.

Mesosphere uses ZooKeeper for coordination, so the templates expect a security group (granting ZK access) and a ZK node discovery URL (exposed by Exhibitor) to be passed in.

Note that this template must be used with Amazon VPC. New AWS accounts automatically use VPC, but if you have an old account and are still using EC2-Classic, you'll need to modify this template or (recommended) make the switch.

##Usage

1. Clone the repository

git clone https://github.com/gboda/AWS-CloudFormationTemplates.git
2. Create an Admin security group

This is a VPC security group containing access rules for cluster administration, and should be locked down to your IP range, a bastion host, or similar. This security group will be associated with the Mesosphere servers.

Inbound rules are at your discretion, but you may want to include access to:

22 [tcp] - SSH port
5050 [tcp] - Mesosphere Master port
8080 [tcp] - Marathon port
3. Set up ZooKeeper

You can use the instructions and template at mbabineau/cloudformation-zookeeper, or you can use an existing cluster.

We'll need two things:

ExhibitorDiscoveryUrl - a URL that returns a list of active ZK nodes. The expected format is that of Exhibitor's /cluster/list call. Example response:
{
    "servers": [
        "zk1.mydomain.com",
        "zk2.mydomain.com",
        "zk3.mydomain.com"
    ],
    "port": 2181
}
ZkClientSecurityGroup - a security group that grants access to the ZooKeeper servers
If you used the aforementioned template, you can simply copy the stack's outputs.

4. Upload the templates to S3

Upload Mesosphere-master.json and Mesosphere-slave.json to an S3 bucket:

aws s3 cp Mesosphere-master.json s3://mybucket/
aws s3 cp Mesosphere-slave.json s3://mybucket/
You'll need to pass the URLs for each as stack parameters. Note the URL should be formatted as https://s3.amazonaws.com/<bucket>/<key>, and the files do not need to be made public.

5. Launch the stack

Launch the stack via the AWS console, a script, or aws-cli.

See Mesosphere.json for the full list of parameters, descriptions, and default values.

Example using aws-cli:

aws cloudformation create-stack \
    --template-body file://Mesosphere.json \
    --stack-name <stack> \
    --capabilities CAPABILITY_IAM \
    --parameters \
        ParameterKey=KeyName,ParameterValue=<key> \
        ParameterKey=ExhibitorDiscoveryUrl,ParameterValue=<url> \
        ParameterKey=ZkClientSecurityGroup,ParameterValue=<sg_id> \
        ParameterKey=VpcId,ParameterValue=<vpc_id> \
        ParameterKey=Subnets,ParameterValue='<subnet_id_1>\,<subnet_id_2>' \
        ParameterKey=AdminSecurityGroup,ParameterValue=<sg_id> \
        ParameterKey=MesosphereMasterTemplateUrl,ParameterValue=<url> \
        ParameterKey=MesosphereSlaveTemplateUrl,ParameterValue=<url>
4. Access the cluster

---

Once the stack has been provisioned, visit the public-facing ELB created by the stack. You can find the DNS address by checking the stack's Outputs.

The ELB exposes two endpoints:

http://<public-elb>:5050/ for Mesosphere
http://<public-elb>:8080/ for Marathon
Note: You will need to do this from a location with access granted by AdminSecurityGroup
