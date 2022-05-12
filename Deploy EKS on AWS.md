## Create Kubernetes deployment on AWS.
## EKS LAB with Nodejs backend. 
---

### Create an AWS User Account

Login in to AWS and the IAM console. Create a new users with the name "Workshop" and select "AWS Management Console Access" and custom password and uncheck "require password reset". Then select "next:Permissions".
Set Permissions:
Select "Attach existing policies directly" and check policy "AdministratorAccess", then select "Next:Preview"
Review:
Review and Create user. Copy the console login URL and save it for later, also save the Download file.csv and these are the credentials that should be kept safe.  

Login now with the login URL you save above. Do not login in with the root user. 


### Create Workspace:
Select the region closest to you that has "Cloud9" supported. You can use the link to find out if you region has Cloud9.
https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/

In the Console select "Cloud9".  
Select "Create environment", call it "eksworkshop", then  click next.
Choose t3.small instance type, the rest of the defaults are fine, so click "Create environment". Start the "Terminal" by selecting the green circle with a plus sign in it. 

The EC2 instance root volume that it create does not have enough disk space so you need to add more. copy and paste and run the commands below in to the terminal. 

```
pip3 install --user --upgrade boto3
export instance_id=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
python -c "import boto3
import os
from botocore.exceptions import ClientError 
ec2 = boto3.client('ec2')
volume_info = ec2.describe_volumes(
    Filters=[
        {
            'Name': 'attachment.instance-id',
            'Values': [
                os.getenv('instance_id')
            ]
        }
    ]
)
volume_id = volume_info['Volumes'][0]['VolumeId']
try:
    resize = ec2.modify_volume(    
            VolumeId=volume_id,    
            Size=30
    )
    print(resize)
except ClientError as e:
    if e.response['Error']['Code'] == 'InvalidParameterValue':
        print('ERROR MESSAGE: {}'.format(e))"
if [ $? -eq 0 ]; then
    sudo reboot
fi
```

### Install Kubectl:

```
sudo curl --silent --location -o /usr/local/bin/kubectl \
   https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl

sudo chmod +x /usr/local/bin/kubectl
```

### Update awscli

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### Install jq, envsubst (from GNU gettext utilities) and bash-completion
```
sudo yum -y install jq gettext bash-completion moreutils
```

### Install yq for yaml processing

```
echo 'yq() {
  docker run --rm -i -v "${PWD}":/workdir mikefarah/yq "$@"
}' | tee -a ~/.bashrc && source ~/.bashrc
```

### Verify the binaries are in the path and executable

```
for command in kubectl jq envsubst aws
  do
    which $command &>/dev/null && echo "$command in path" || echo "$command NOT FOUND"
  done
```

### Enable kubectl bash_completion

```
kubectl completion bash >>  ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```

### set the AWS Load Balancer Controller version
```
echo 'export LBC_VERSION="v2.4.1"' >>  ~/.bash_profile
echo 'export LBC_CHART_VERSION="1.4.1"' >>  ~/.bash_profile
.  ~/.bash_profile
```

---

### Create IAM Role for Workspace.
Got to the  IAM console and select "Roles", select "Create Role"
Confirm that AWS service and EC2 are selected, then click Next: Permissions to view permissions.
Confirm that AdministratorAccess is checked, then click "Next: Tags".
Take the defaults, and click "Next: Review" to review.
Enter eksworkshop-admin for the Name, and click Create role.

### Attach role to Workspace
Click the grey circle button (in top right corner) and select **Manage EC2 Instance**
This will take you to the EC2 Console. 
Select the instance, then choose Actions / Security / Modify IAM Role.
Choose eksworkshop-admin from the IAM Role drop down, and select Save.

### Update IAM Settings for your Workspace
To ensure temporary credentials aren’t already in place we will remove any existing credentials file as well as disabling **AWS managed temporary credentials**:

```
aws cloud9 update-environment  --environment-id $C9_PID --managed-credentials-action DISABLE
rm -vf ${HOME}/.aws/credentials
```

Using CLI configure the AWS default region. 
```
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
export AZS=($(aws ec2 describe-availability-zones --query 'AvailabilityZones[].ZoneName' --output text --region $AWS_REGION))
```

Check if AWS_REGION is set to desired region.
```
test -n "$AWS_REGION" && echo AWS_REGION is "$AWS_REGION" || echo AWS_REGION is not set
```

Save this in to Bash Profile.
```
echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
echo "export AZS=(${AZS[@]})" | tee -a ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure get default.region
```

### Validate the IAM role
Validate that the Cloud9 IDE is using the correct IAM role.
```
aws sts get-caller-identity --query Arn | grep eksworkshop-admin -q && echo "IAM role valid" || echo "IAM role NOT valid"
```

If the IAM role is not valid, **DO NOT PROCEED**. Go back and confirm the steps on this page.

---

### Clone the service Repos.
```
cd ~/environment git clone https://github.com/aws-containers/ecsdemo-frontend.git git clone https://github.com/aws-containers/ecsdemo-nodejs.git git clone https://github.com/aws-containers/ecsdemo-crystal.git
```

---

### Create AWS KMS Custom managed key(CMK)
Create a CMK for the EKS cluster to use when encrypting your Kubernetes secrets:
```
aws kms create-alias --alias-name alias/eksworkshop --target-key-id $(aws kms create-key --query KeyMetadata.Arn --output text)
```

Retrieve the ARN of the CMK to input into the create cluster command.
```
export MASTER_ARN=$(aws kms describe-key --key-id alias/eksworkshop --query KeyMetadata.Arn --output text)
```

Set the MASTER_ARN environment variable to make it easier to refer to the KMS key later.Now, save the MASTER_ARN environment variable into the bash_profile
```
echo "export MASTER_ARN=${MASTER_ARN}" | tee -a ~/.bash_profile
```

---

### Launch eksctl 
Download the eksctl binary
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv -v /tmp/eksctl /usr/local/bin
```

Confirm the eksctl command works:

```
eksctl version
```

Enable eksctl bash-completion

```
eksctl completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```

---
### Launch EKS
Make sure that you have validated the IAM role used by Cloud9. To check this run and make sure it contains *ekworkshop-admin* in ARN. go back to validate IAM role if incorrect.

```
aws sts get-caller-identity
```

### Create an EKS cluster
eksctl version must be 0.38.0 or above to deploy EKS 1.19, click here to get the latest version.

```
cat << EOF > eksworkshop.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksworkshop-eksctl
  region: ${AWS_REGION}
  version: "1.19"

availabilityZones: ["${AZS[0]}", "${AZS[1]}", "${AZS[2]}"]

managedNodeGroups:
- name: nodegroup
  desiredCapacity: 3
  instanceType: t3.small
  ssh:
    enableSsm: true

# To enable all of the control plane logs, uncomment below:
# cloudWatch:
#  clusterLogging:
#    enableTypes: ["*"]

secretsEncryption:
  keyARN: ${MASTER_ARN}
EOF
```

Use the file you created as the input for the eksctl cluster creation

```
eksctl create cluster -f eksworkshop.yaml
```

Launching EKS and all the dependencies will take approximately 15 minutes. You can check on it's progress through "Cloudformation"

---

### Test you cluster
Check node are running, you should 3 nodes, we know we have authenticated correctly

```
kubectl get nodes
```

### Export the Worker Role Name for use later.

```
STACK_NAME=$(eksctl get nodegroup --cluster eksworkshop-eksctl -o json | jq -r '.[].StackName')
ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name $STACK_NAME | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')
echo "export ROLE_NAME=${ROLE_NAME}" | tee -a ~/.bash_profile
```

EKS Cluster Created!!!!!

---

## Deploy Microservices
### Deploy NodeJS backend.

Make sure you have cloned the Repos listed above.

```
cd ~/environment/ecsdemo-nodejs
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
```
 
 Watch the progress status
 
 ```
 kubectl get deployment ecsdemo-nodejs
 ```

### Deploy Crystal Backend API

```
cd ~/environment/ecsdemo-crystal
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml

```

 Watch the progress status
 
 ```
 kubectl get deployment ecsdemo-crystal
 ```

check to see if the loadbalancer exists

```
aws iam get-role --role-name "AWSServiceRoleForElasticLoadBalancing" || aws iam create-service-linked-role --aws-service-name "elasticloadbalancing.amazonaws.com"
```

---

### Deploy Frontend Service

Deploy Ruby Frontend

```
cd ~/environment/ecsdemo-frontend
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
```

Watch the progress
```
kubectl get deployment ecsdemo-frontend
```


### Find Service Address

To get the type:Loadbalancers address:

```
kubectl get service ecsdemo-frontend -o wide
```

To get it in a json format:

```
ELB=$(kubectl get service ecsdemo-frontend -o json | jq -r '.status.loadBalancer.ingress[].hostname')

curl -m3 -v $ELB

```

---

### Scaling  Backend Services

Check what deployments are running 

```
kubectl get deployments
```

Scale services up:

```
kubectl scale deployment ecsdemo-nodejs --replicas=3
kubectl scale deployment ecsdemo-crystal --replicas=3
```

Check what deployments are running 3 replicas

```
kubectl get deployments
```

---

### Scale Frontend Services

Scale services up by 3:
```
kubectl get deployments
kubectl scale deployment ecsdemo-frontend --replicas=3
kubectl get deployments
```

### Cleanup applications

```
cd ~/environment/ecsdemo-frontend
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml

cd ~/environment/ecsdemo-crystal
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml

cd ~/environment/ecsdemo-nodejs
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml
```



