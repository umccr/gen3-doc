# Gen3 Cloud Native EKS Setup

These are UMCCR notes on getting up and running of the [Gen3 Cloud Automation][1] deployment. Gen3 comprises multiple software components; deploy and run in Microservices fashion. The focus here is to up and run those _mandatory_ Gen3 services or, baseline operation of the platform.

### Prerequisite

> Please see [AWS](AWS.md) notes.

## Infrastructure Setup

It follows along the [csoc-free-commons-steps][2] described in GitHub. It is a quick run through the steps we needed to perform to get that initial setup up and running. It also contains pointers for things to look out for, that otherwise may get you into trouble during setup or further down the road.

### Part 1: Admin VM

#### AWS EC2 admin server

_UMCCR setup summary:_ 

Gen3 deployment/management scripts depend on the `$HOME` variable, hence it's easiest to setup Gen3 from the users home directory. We currently create EC2 volume snapshots in order to preserve the Gen3 setup/configuration. To keep these snapshots small and facilitate reuse across instances (which is difficult with root volumes), we add a second volume to the instance. We generate a new OS user with a custom `$HOME` directory in order to split this data from the default root volume.

- Ubuntu 18.04 server (issues with newer versions or other OS)
- t2.medium
- min 15GB disk
- hibernation enabled (in order to be able to shut down the admin VM when not needed. Longer term plan to restore admin VM from scratch from backed up configuration/data)
- admin like IAM role (`gen3-admin-vm-instance-role`)
- no SSH key -> use SSM
- SG (`gen3-sg`) could be restricted to Gen3 internal SSH access only
- Gen3 config/data volume in addition to the default root volume (see below)
- custom user `gen3-user`

We have CDK stack for aforementioned setup as follows:
- https://github.com/umccr/infrastructure/tree/master/cdk/apps/gen3_admin_vm

We also use our fork with regional and few fixes:
- https://github.com/umccr/cloud-automation/tree/umccr

#### Bootstrap admin server

```bash
# working outside the users $HOME caused issues (as it is assumed in various scripts)
cd $HOME

# we retain our fork with 'umccr' branch for any changes
git clone https://github.com/umccr/cloud-automation.git
cd cloud-automation
git checkout umccr

export GEN3_HOME="$HOME/cloud-automation"
export GEN3_NOPROXY='no'

# be careful to retain the env vars when running the setup script
bash cloud-automation/gen3/bin/kube-setup-workvm.sh

# double check the entries created in the .bashrc
vim ${HOME}/.bashrc
source ${HOME}/.bashrc

# prepare the AWS profiles
# NOTE: remove the credential_source, as it caused issues with Terraform 12 AWS provider (CredentialRequiresARNError)

echo "[default]
output = json
region = ap-southeast-2

[profile cdistest]
output = json
region = ap-southeast-2" > ~/.aws/config
```

### Part 2: Common Stack

- At the point, you need to provision the following resources:
  - Decide on domain name to use e.g. `gen3.cloud.dev.umcc.org` 
  - Create Route53 hosted zone if you are sub-zoning, if any
  - Create ACM certificate for your chosen domain name


- Note that in the following terraform common stack setup, it will take the stack name `umccr-test` as a name to create a VPC. Make sure this common stack name (VPC name) is **no longer than 14 characters**, as other resource names (specifically the AWS ElasticSearch domain name) are derived from it and, have size limitations attached.

```bash
gen3 workon cdistest umccr-test

gen3 cd

# update/complete config.tfvars according to documentation
vim config.tfvars

gen3 tfplan
gen3 tfapply
# may have to repeat (TF may run into concurrency issues)

# backup the secrets/config for later use
cp umccr-test_output/* $HOME/Gen3Secrets/
```

### Part 3: Kubernetes Cluster

- Generate new key pair through `EC2 Console > Network & Security > Key Pairs` with key name, e.g. `gen3-cloud-kube-worker` and, save the private key. This will be needed in `config.tfvars` and, later when we want to ssh into Worker nodes.

```bash
gen3 workon cdistest umccr-test_eks
gen3 cd

# update/complete config.tfvars according to documentation
vim config.tfvars

gen3 tfplan
gen3 tfapply

# backup the secrets/config for later use
cp umccr-test_output_EKS/kubeconfig $HOME/Gen3Secrets/
```

### Part 4: ElasticSearch

```bash
gen3 workon cdistest umccr-test_es
gen3 cd

# update/complete config.tfvars according to documentation
vim config.tfvars

gen3 tfplan
gen3 tfapply
```

### Part 5: Deployment Manifest

Prepare deployment manifest for the Gen3 instance, e.g. `gen3.cloud.dev.umccr.org` and, create `cdis-manifest` repo as follows.

- https://github.com/umccr/cdis-manifest

Next, on Admin VM, clone this manifest repo as follows.

```bash
cd $HOME
git clone https://github.com/umccr/cdis-manifest.git
```

Note: if you are new to [GitOps concept](https://www.google.com/search?q=What+is+GitOps%3F), it is recommended to do some reading around the concept.

### Part 6: Gen3 Services

As mentioned in the [Gen3OnK8s][3] docs, Kubernetes uses a manifest file to define which services to run and how to configure them. An example can be found [here][4]. Then, we call roll all command to provision all Gen3 services that defined in deployment manifest.

```bash
source ${HOME}/.bashrc

kubectl apply -f ${HOME}/Gen3Secrets/00configmap.yaml

gen3 roll all
```

#### Checking Kubernetes services

By now you should have a basic Gen3 cloud deployment running. You should be able to run `kubectl` commands to inspect your cluster.

Examples:
```bash
kubectl get nodes
kubectl get pods
kubectl get service
kubectl get deploy
kubectl describe configmap -n kube-system aws-auth

kubectl logs -f fence-deployment-b6cf954d9-jt9xt -c fence
```


## Configuring Gen3 Services

Once the Kubernetes cluster is up and running, we configure Gen3 services that is deployed into Kubernetes cluster. You may add or remove Gen3 components that fit for your use case. 

The following are break-out notes on configuring each Gen3 services. You may be repeating these steps as if needed. 

### Domain Name

The following command should give you the hostname you can direct your browser to in order to access your Gen3 service. We map this to our custom domain (e.g. `gen3.cloud.dev.umccr.org`) using AWS `Route53`.

```bash
kubectl get service revproxy-service-elb -o json | jq -r .status.loadBalancer.ingress[].hostname
```

### Fence

Useful commands
```bash
kubectl get secrets
kubectl get secrets fence-config
kubectl describe secrets fence-config

# decode (and save) current fence config
cd $HOME/Gen3Secrets
ll apis_configs/fence-config.yaml
gen3 secrets decode fence-config > apis_configs/fence-config.yaml

# verify/check yaml
python3 -c 'import yaml, sys; print(yaml.safe_load(sys.stdin))' < apis_configs/fence-config.yaml
python3 -c 'import yaml, sys; yaml.safe_load(sys.stdin)' < apis_configs/fence-config.yaml

# update fence after configuration changes
kubectl delete secret fence-config && gen3 kube-setup-fence

# get the location of the current user.yaml
kubectl get configmap manifest-global -o=jsonpath='{.data.useryaml_s3path}'

# refresh user.yaml config and check logs
gen3 job run usersync
gen3 job logs usersync
```


### Tube

The Tube service translates between the hierarchical DB data model and seach optimised flat indexes in ElasticSearch. See [here][7] for more details.

```bash
# roll specific service as usual
gen3 roll tube

kubectl logs tube-deployment-yyyyyyyyy-xxxxx
kubectl get pod

# may require manual fixing of the Tupe service:
# enter the service pod with an interactive session
kubectl exec -it tube-deployment-yyyyyyyyy-xxxxx -- bash
curl http://esproxy-service:9200/
curl http://esproxy-service:9200/_cat/indices

ls -l
/usr/local/bin/python -m pip install --upgrade pip
pip list | grep gdcdictionary
pip install gdcdictionary
python run_config.py && python run_etl.py

# on the Admin VM update the etlMapping and refresh the service
  (update etlMapping)
  kubectl delete configmap etl-mapping
  gen3 kube-setup-secrets
  kubectl describe cm etl-mapping
  gen3 roll tube

# the service should work and produce reasonable output
root@tube-deployment-yyyyyyyyy-xxxxx:/tube# curl http://esproxy-service:9200/_cat/indices
green open file-array-config_0 Ujv_WY3lRZKXsQpLPp9p8A 5 1  1 0   8.5kb   4.2kb
green open .kibana_1           MJ8NAf1eSZaky3MIHofIwg 1 1  0 0    522b    261b
green open etl_0               g7-E2L0rRD2Z4kGVb4S-qg 5 1 10 0 303.1kb 151.5kb
green open etl-array-config_0  YQZWGu6XTp2YLpmTNRURJg 5 1  1 0   8.8kb   4.4kb
green open file_0              Q4vU-nxSSTCOfSzci5H6_w 5 1 60 0 847.3kb 423.6kb
****
```


### ElasticSearch & Guppy

Following this guide setup your etlMapping and gitops config.
```bash
cd cdis-manifest/gen3.cloud.dev.umccr.org/
vim etlMapping.yaml

mkdir portal
cd portal
vim gitops.json
```

NOTE: the Guppy service may require existing indexes and fail on a clean setup. If so, disable it initially until the first data is loaded.

```bash
vim cdis-manifest/gen3.cloud.dev.umccr.org/manifest.json

gen3 reset
```

Once data is loaded and ElasticSearch indexes are available add the Guppy service back into the manifest and deploy it.
```bash
kubectl get pod
gen3 roll guppy

kubectl describe cm manifest-guppy
kubectl logs guppy-deployment-yyyyyyyyyy-xxxxx
```


### SSJDISPATCHER

This service is required for the data (file) upload.

Have a look at the [kube setup][5] and the [data upload][6] for more details.

```bash
gen3 kube-setup-ssjdispatcher --help

# note this 'auto' mode will create another upload bucket again!  
gen3 kube-setup-ssjdispatcher auto

# perhaps we pass-in data bucket and SQS queue created from "Part 2: Common Stack", e.g.
gen3 kube-setup-ssjdispatcher umccr-test-data-bucket https://sqs.ap-southeast-2.amazonaws.com/012345678912/umccr-test-data-bucket_data_upload

kubectl get secret ssjdispatcher-creds
kubectl describe secret ssjdispatcher-creds
gen3 secrets decode ssjdispatcher-creds | less

# fence-bot IAM user credentials is in Common Stack terraform output
gen3 workon cdistest umccr-test
gen3 cd
gen3 tfoutput

    "fence-bot_user_id": {
        "sensitive": false,
        "type": "string",
        "value": "EXAMPLEIDEXAMPLEID"
    },
    "fence-bot_user_secret": {
        "sensitive": false,
        "type": "string",
        "value": "examplesecretexamplesecretexamplesecret"
    },

# use fence-bot IAM user cred to configure upload bucket in fence config
vi $HOME/Gen3Secrets/apis_configs/fence-config.yaml
```


### Portal Config

- Create or modify in portal config directory e.g. `cdis-manifest/gen3.cloud.dev.umccr.org/portal/`

```
gitops.css
gitops-logo.png
gitops-favicon.ico
gitops.json
```

- Tune CSS according to your need
- Make sure, in `manifest.json` **_global_** block > **_portal_app_** has `gitops`. Not `dev` nor any other value; example see [this line](https://github.com/umccr/cdis-manifest/blob/master/gen3.cloud.dev.umccr.org/manifest.json#L222)
- Then, delete Portal config and setup again, like so:
```
kubectl delete secret portal-config && gen3 kube-setup-portal
```

### CILogon

See [cilogon](../cilogon)


[1]: https://github.com/uc-cdis/cloud-automation
[2]: https://github.com/uc-cdis/cloud-automation/blob/master/doc/csoc-free-commons-steps.md
[3]: https://github.com/uc-cdis/cloud-automation/blob/master/doc/gen3OnK8s.md
[4]: https://github.com/uc-cdis/cdis-manifest/blob/master/caninedc.org/manifest.json
[5]: https://github.com/uc-cdis/cloud-automation/blob/master/doc/kube-setup-ssjdispatcher.md
[6]: https://github.com/uc-cdis/cloud-automation/blob/master/doc/Gen3-data-upload.md
[7]: https://github.com/uc-cdis/cloud-automation/blob/master/kube/services/tube/README.md
