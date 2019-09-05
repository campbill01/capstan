# Amazon Web Services Featuring Spinnaker on EKS

This Guide Deploy Spinnaker on EKS with some AWS specific IaaS integrations for other AWS services

## Limitations

The following are known limitations

- Testing occured in the first few regions that recieved EKS initially
- This solution assumes `us-west-2`
- Solution assumes 3 AZ(s)

## What Capstan will Create

Capstan on AWS will build the following solution shown in the illustration:


This includes:
- A New VPC (subnets, gatways, routing tables, etc)
- Security Groups in said VPC
- Iam Roles / Instance Profiles
- Iam User for Terraform
- EKS Cluster
- A bastion that contains tools and be used as a ssh tunnel endpoint 
  - The security group created allows access only from your Workstation's IP address.
  - This tools instance performs addition EKS cluster configuration like install helm packagaes and automatically deploying spinnaker via [halyard]()


## Let's Get Started

*These steps already assume that you can use the AWS CLI from your workstation and can run commands aginst your AWS account with the AWS CLI.*

### Additional Installation

Normally, the only tools you will need on your computer will be terraform, git, and IaaS tools (here the AWS CLI). However, for authentication with EKS from your workstation you will also need the `aws-iam-authenticator`. You can read about the installation process [here]().

If you are on a mac you can download it here
`curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.11.5/2018-12-06/bin/darwin/amd64/aws-iam-authenticator`

make sure you make it executable
`chmod +x aws-iam-authenticator`

`sudo mv aws-iam-authenticator /somewhere/in/your/path.`

To make sure that 


### Advanced Set-up
TBD,  uses DNS/TLS/Oauth2

### Set-up Terraform

To configure terraform, we will:
- Use AWS Native IaaS mark-up terraform to configure and AWS Account for terraform
- Validate that terraform can modify said AWS account

#### Deploy Cloudformation

While logged in as a user that can manipulate AWS IAM with CloudFormation, you will apply the cloudformation template located [here](./cloudformation/tf_infra.yml). 

The cloudformation template will create an IAM user for Terraform and an IAM role for terraform. 

On the output tab you should see and entry for
- AccessKey
- SecretAccessKey
- TerraformRole (capstan-terraform-role-assumed)

Terraform will use these values to manipulate your aws account.

#### Get new key pair

Since we will be using/accessing EC2 we need to create a new key pair. We will use this keypair to use the tools/tunnel instance we created. To learn how to create an EC2 key pair see [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html). You will create the key pair in the `us-west-2`. The name of the key pair should be `capstan`. If you have another key or want to use another name you will have to change it in the `variables.tf` file. When the pem file downloads place in the same folder as terraform files. 

```
variable "ec2_key" {
  default = "capstan"
  type    = "string"
}

variable "ec2_key_file" {
  default = "capstan.pem"
  type    = "string"
}
```

#### Validate your setup

Now it is time to make sure that your AWS account is ready to go. you will take values from the previous deployed cloud formation and use them as values for terraform and aws-iam-authenticator. In your bash compliant shell you will create some environment variables for the terraform as shown:


```
export AWS_ACCESS_KEY_ID="anaccesskey"
export AWS_SECRET_ACCESS_KEY="asecretkey"
export AWS_DEFAULT_REGION="avalidregion"
export TF_VAR_aws_account_id="<insert account number>"
```

In the same shell type the following command:

`terraform init`
This command will parse the terraform and identifiy any modules that need to downloaded. There are some courtesy error checks performed BUT nothing context specific. If there are no errors execute the next command.

`terraform plan`

This create a listing of all the AWS resources that will be created by the terraform project. This should not generate errors.



## Finally, You are ready to begin

## Create the environment
`terraform apply`

This will ask for confirmation, only a yes answer will proceed.
After entering yes, terraform will create the resources in AWS.
This will take about 20 minutes.

## Accessing the Environment
Assuming that you have 

### Kubectl

This solution  downloaded a version of kubectl to your machine. It is located in the `aws/terraform` folder. In your shell in the terraform folder

`./kubectl --kubeconfig=generated-kube.conf get pods --namespace spinnaker`

This command will list all the pods that spinnaker deployed. The ready column in the output should have 1/N for all services. 

 
### Advanced Access

If you set-up advanced access, you can skip the following two sections
- Observavility
- Spinnaker

Use the DNS names referenced in the Advanced Access Set-up

### Observability

#### Logs
This solution installed EFK. In another terminal window navigate to the `aws/terraform` folder and run the following commands

- `export POD_NAME=$(./kubectl --kubeconfig=generated-kube.conf get pods  -l "app=kibana,release=kibana" -o jsonpath='{.items[0].metadata.name}')`
  - That command looked up the Kibana Web App
- `./kubectl --kubeconfig=generated-kube.conf port-forward  $POD_NAME 5601:5601`
  - This created a port forward between your machine and the Kibana App

You should then be able to navigate in your browser in "in-cognito mode" to:
`http://localhost:5601/app/kibana`

You will be prompted to create an index pattern for Kibana. If you don't see something like `kubernetes_cluster-yyyy.mm.dd` that means the index for the day was not created yet. Come back later.

#### Metrics/Tracing
TBD

### Access Spinnaker

To access spinnaker you are going to ssh into the tools/tunnel instance and use halyard to facilitate port forwarding to your work station. In a new shell navigate to `aws/terraform`

`ssh -i [PEM_NAME].pem ubuntu@[IP_or_DNS_of_Instance] -L 9000:localhost:9000 -L 8084:localhost:8084`

Once you have ssh into the instance

`hal deploy connect`

When that command succedes open your browser in an incognito window and type

`http://localhost:9000`

You should see the initial spinnaker UI

You are in!


### Adding Jenkins

TBD

## Destroy everything

If you run `terraform destroy` after use of this environment you may find that some items can't be destroyed. These will ussually be
- S3 buckets created
- Network

The leading cause for S3 buckets is that terraform does not delete buckets with contents. The Bucket for Spinnaker will have connect

Regarding network, this most often occurs after deploying apps to K8 that will create addition network resources in the VPC. 

Regardless, you have to manuallly remove items using the AWS Console/CLI and then run `terraform destory`