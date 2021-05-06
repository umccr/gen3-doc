# Gen3 Cloud Native EKS Setup

Follow general procedure outlined here:
https://github.com/uc-cdis/cloud-automation/blob/master/doc/csoc-free-commons-steps.md


These are notes on the UMCCR specific setup. It is follows closely the instructions set out in the above mentioned documentation, with minor adjustments where they were needed.


## Part 1 Admin VM

AWS EC2 admin server

UMCCR setup:

- Ubuntu 18.04 server (issues with newer versions or other OS)
- t2.meduim
- min 10GB disk (should probably be a bit more)
- hibernation enabled (could switch to creating new instances and recover data/config)
- admin like IAM role (`gen3-admin-vm-instance-role`)
- no SSH key -> use SSM
- SG (`gen3-sg`) to allow inbound SSH


Instance setup
```bash
# work under the system's ubuntu user
sudo su ubuntu
# make sure the user has full access/control of required folders
sudo chown ubuntu:ubuntu ~/.aws ~/.local
```


Follow the Gen3 setup steps
```bash
# working outside the users $HOME caused issues (as it is assumed in various scripts)
cd $HOME
git clone https://github.com/uc-cdis/cloud-automation.git
export GEN3_HOME="$HOME/cloud-automation"
export GEN3_NOPROXY='no'
# be careful to retain the env vars when running the setup script
sudo -E bash cloud-automation/gen3/bin/kube-setup-workvm.sh

# Double check the entries created in the .bashrc
vim ${HOME}/.bashrc
source ${HOME}/.bashrc

# prepare the AWS profiles
# NOTE: remove the credential_source, as it caused issues with Terraform 12 AWS provider (CredentialRequiresARNError)

echo "[default]
output = json
region = ap-southeast-2
#credential_source = Ec2InstanceMetadata

[profile cdistest]
output = json
region = ap-southeast-2" > ~/.aws/config
```



## Part 2 Start Gen3 - Infrastructure

NOTE: make sure the Gen3 env name is no longer than 14 characters, as other resouece names (specifically the AWS Elastic Search domain) are derived from it and have size limitations attached.
```bash
# should be no longer than 14 characters
gen3 api environment
```

```bash
gen3 workon cdistest umccr-commons-test
# may have to create S3 bucket manually if TF raises issues
# e.g. aws s3 mb s3://cdis-state-ac064933140851-gen3
gen3 cd

# Fix squid issue cross account issue (see: https://github.com/uc-cdis/cloud-automation/pull/1507)

# TEMP ONLY: Remove CloudTrail setup from TF stack (currently denied on UoM AWS)
rm ${GEN3_HOME}/tf_files/aws/modules/upload-data-bucket/cloud-trail.tf

# update/complete config.tfvars according to documentation

gen3 tfplan
gen3 tfapply
# may have to repeat (TF may run into concurrency issues)

# backup the secrets/config for later use
cp umccr-commons-test_output/* $HOME/Gen3Secrets/
```


## Part 3 Kubernetes Cluster - Infrastructure

```bash
gen3 workon cdistest umccr-commons-test_eks
gen3 cd

# update/complete config.tfvars according to documentation
# Fix hard coded availability zones in TF EKS module (see: https://github.com/uc-cdis/cloud-automation/pull/1573)

gen3 tfplan
gen3 tfapply

# backup the secrets/config for later use
cp umccr-commons-test_output_EKS/kubeconfig $HOME/Gen3Secrets/
```


## Part 4 Kubernetes Services

```bash
# Create manifest.json following documentation
mkdir -p ${HOME}/cdis-manifest/gen3.cloud.dev.umccr.org

# adjust ${HOME}/.bashrc (see bash-extensions.txt)
source ${HOME}/.bashrc
kubectl apply -f ${HOME}/Gen3Secrets/00configmap.yaml
kubectl get nodes
gen3 roll all
```

## Check Kubernetes services

```bash
kubectl get nodes
kubectl get pods
kubectl get service
kubectl get deploy
kubectl describe configmap -n kube-system aws-auth

kubectl logs -f fence-deployment-b6cf954d9-jt9xt -c fence
...
```
