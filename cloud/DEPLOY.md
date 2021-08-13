# Gen3 Cloud Native EKS Setup

These are notes on the UMCCR specific setup of the [Gen3 cloud deployment][1].

- [Gen3 Cloud Native EKS Setup](#gen3-cloud-native-eks-setup)
	- [Basic setup](#basic-setup)
		- [Part 1 Admin VM](#part-1-admin-vm)
		- [Part 2 Start Gen3 - Infrastructure](#part-2-start-gen3---infrastructure)
		- [Part 3 Kubernetes Cluster - Infrastructure](#part-3-kubernetes-cluster---infrastructure)
		- [Part 4 Kubernetes Services](#part-4-kubernetes-services)
		- [Check Kubernetes services](#check-kubernetes-services)
	- [Additional setup](#additional-setup)
		- [Elastic Search & Guppy](#elastic-search--guppy)
		- [SSJDISPATCHER](#ssjdispatcher)
		- [Fence](#fence)
		- [Tube](#tube)
		- [Portal Config](#portal-config)
		- [CILogon](#cilogon)


## Basic setup

The basic setup follows along the [csoc-free-commons-steps][2] described in GitHub. It is a quick run through the steps we needed to perform to get that initial setup up and running. It also contains pointers for things to look out for, that otherwise may get you into trouble during setup or further down the road.

NOTE: this is not our fully featured production setup. For more information on that, see the sections in [Additional setup](#additional-setup) below.


### Part 1 Admin VM

AWS EC2 admin server

UMCCR setup:

- Ubuntu 18.04 server (issues with newer versions or other OS)
- t2.meduim
- min 15GB disk
- hibernation enabled (in order to be able to shut down the admin VM when not needed. Longer term plan to restore admin VM from scratch from backed up configuration/data)
- admin like IAM role (`gen3-admin-vm-instance-role`)
- no SSH key -> use SSM
- SG (`gen3-sg`) could be restricted to Gen3 internal SSH access only
- Gen3 config/data volume in addition to the default root volume (see below)
- custom user `gen3-user`

Gen3 deployment/management scripts depend on the `$HOME` variable, hence it's easiest to setup Gen3 from the users home directory. We currently craete EC2 volume snapshots in order to preserve the Gen3 setup/configuration. To keep these snapshots small and facilitate reuse across instances (which is difficult with root volumes), we add a second volume to the instance. We generate a new OS user with a custom `$HOME` directory in order to split this data from the default root volume.




Follow the Gen3 setup steps
```bash
# working outside the users $HOME caused issues (as it is assumed in various scripts)
cd $HOME
# Note: Terraform updated their certificates which means you may run into issues with older source versions (0.11.14).
#       We tried the master (at 6cbb6035), which contained a TF version update (0.11.15) to fix this issue
git clone https://github.com/uc-cdis/cloud-automation.git
export GEN3_HOME="$HOME/cloud-automation"
export GEN3_NOPROXY='no'
# be careful to retain the env vars when running the setup script
sudo -E bash cloud-automation/gen3/bin/kube-setup-workvm.sh

# Double check the entries created in the .bashrc
# NOTE: make sure the Gen3 env name (VPC name) is no longer than 14 characters, as other resouece names (specifically the AWS #       Elastic Search domain) are derived from it and have size limitations attached.
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



### Part 2 Start Gen3 - Infrastructure

```bash
gen3 workon cdistest umccr-test
# may have to create S3 bucket manually if TF raises issues
# e.g. aws s3 mb s3://cdis-state-ac064933140851-gen3
gen3 cd

# Fix squid issue cross account issue (see: https://github.com/uc-cdis/cloud-automation/pull/1507)

# TEMP ONLY: Remove CloudTrail setup from TF stack (currently denied on UoM AWS)
rm ${GEN3_HOME}/tf_files/aws/modules/upload-data-bucket/cloud-trail.tf

# update/complete config.tfvars according to documentation
vim config.tfvars

gen3 tfplan
gen3 tfapply
# may have to repeat (TF may run into concurrency issues)

# backup the secrets/config for later use
cp umccr-test_output/* $HOME/Gen3Secrets/
```



### Part 3 Kubernetes Cluster - Infrastructure

```bash
gen3 workon cdistest umccr-test_eks
gen3 cd

# update/complete config.tfvars according to documentation
# Fix hard coded availability zones in TF EKS module (see: https://github.com/uc-cdis/cloud-automation/pull/1573)
vim config.tfvars

gen3 tfplan
gen3 tfapply

# backup the secrets/config for later use
cp umccr-test_output_EKS/kubeconfig $HOME/Gen3Secrets/
```



### Part 4 Kubernetes Services

```bash
# Create manifest.json following documentation
mkdir -p ${HOME}/cdis-manifest/gen3.cloud.dev.umccr.org

# adjust ${HOME}/.bashrc (see bash-extensions.txt)
source ${HOME}/.bashrc
kubectl apply -f ${HOME}/Gen3Secrets/00configmap.yaml
kubectl get nodes
gen3 roll all
```



### Check Kubernetes services

By now you should have a basic Gen3 cloud deployment running. You should be able to run kubectl commands to inspect your cluster.

Examples:
```bash
kubectl get nodes
kubectl get pods
kubectl get service
kubectl get deploy
kubectl describe configmap -n kube-system aws-auth

kubectl logs -f fence-deployment-b6cf954d9-jt9xt -c fence
...
```


## Additional setup

Once the Kubernetes cluster is up and running additional services can usually be integrated on the Kubernetes level. Exceptions are services that depend on (AWS) infrastructure that is not managed by Kubernetes.
[Here][3] is a quick overview of how Kubernetes is used and configured for Gen3.

Below are examples of services that required additional work.

### Elastic Search & Guppy

A production release of Gen3 typically comes with an Elastic Search component. This is not part of the default setup described in the [csoc-free-commons-steps][2]. It utilises the AWS Elastic Search managed service and hence requires additional infrastructure to be deployed.

We follow the basic steps to add another infrastructure component:
```bash
gen3 workon cdistest umccr-test_es
gen3 cd

# update/complete config.tfvars according to documentation
vim config.tfvars

gen3 tfplan
gen3 tfapply
```

As mentioned in the [gen3OnK8s][3] docs, Kubernetes uses a manifest file to define which services to run and how to configure them. An example can be found [here][4].

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

source ${HOME}/.bashrc

kubectl apply -f ${HOME}/Gen3Secrets/00configmap.yaml

gen3 roll all
gen3 reset
```

Once data is loaded and Elastic Search indexes are available add the Guppy service back into the mainfest and deploy it.
```bash
kubectl get pod
gen3 roll guppy

kubectl describe cm manifest-guppy
kubectl logs guppy-deployment-yyyyyyyyyy-xxxxx
```



The following command should give you the hostname you can direct your browser to in order to access your Gen3 service.
We map this to our custom domain (gen3.cloud.dev.umccr.org) using AWS `Route53`.
```bash
kubectl get service revproxy-service-elb -o json | jq -r .status.loadBalancer.ingress[].hostname
```


### SSJDISPATCHER

This service is required for the data (file) upload.

Have a look at the [kube setup][5] and the [data upload][6] for more details.

```bash
gen3 kube-setup-ssjdispatcher auto

kubectl get secret ssjdispatcher-creds
kubectl describe secret ssjdispatcher-creds
gen3 secrets decode ssjdispatcher-creds | less

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

vi $HOME/Gen3Secrets/apis_configs/fence-config.yaml
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

# check logs
kubectl logs useryaml-xxxxx
gen3 job logs useryaml

# refresh user config and check logs
gen3 job run usersync
kubectl logs usersync-xxxxx fence

gen3 job run useryaml
kubectl logs useryaml-xxxxx
```

### Tube

The Tube service translates between the hierarchical DB data model and seach optimised flat indexes in Elastic Search. See [here][7] for more details.

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

The support for CILogon in the Fence service was implemented by Scott Koranda with Pull Request [#896][8].

NOTE: Although this PR was merged before the current production release (`2021.05`) of Gen3, the release does not include Scott's changes. We therefore selected the `master` version of the Fence container, which does support CILogon (update the Kubernetes manifest).

```bash
# update fence version (if needed)
vim $HOME/cdis-manifest/gen3.cloud.dev.umccr.org/manifest.json
```

Enable CILogon in the Fence config. An example can be found in the [config-default.yaml][9].
```bash
vim $HOME/Gen3Secrets/apis_configs/fence-config.yaml
```

To get the `client_id` and `client_secret` you create a new `OIDC Client` in your COmanage account (`Configuration` menu).
Don't forget to add a LDAP to Claim mapping that maps LDAP Attribute Name `voPersonApplicationUID;app-gen3` to OIDC Claim Name `sub` to get a readable and consistant user name (even if you link multiple identities to your COmanage account).



[1]: https://github.com/uc-cdis/cloud-automation
[2]: https://github.com/uc-cdis/cloud-automation/blob/master/doc/csoc-free-commons-steps.md
[3]: https://github.com/uc-cdis/cloud-automation/blob/master/doc/gen3OnK8s.md
[4]: https://github.com/uc-cdis/cdis-manifest/blob/master/caninedc.org/manifest.json
[5]: https://github.com/uc-cdis/cloud-automation/blob/master/doc/kube-setup-ssjdispatcher.md
[6]: https://github.com/uc-cdis/cloud-automation/blob/master/doc/Gen3-data-upload.md
[7]: https://github.com/uc-cdis/cloud-automation/blob/master/kube/services/tube/README.md
[8]: https://github.com/uc-cdis/fence/pull/896
[9]: https://github.com/uc-cdis/fence/blob/master/fence/config-default.yaml
