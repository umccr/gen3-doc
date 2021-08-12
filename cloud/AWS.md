# AWS Resources

> **DISCLAIMER:** The following are **our best estimate pointer** for those who like to ops Gen3 platform. The following specs and tech stack are _empirical observation_ through our pilot run -- which **may not hold true** for any other variations.

> **NOTE:** Cloud Automation setup may not well fit for those who like to do quick POC/prototype. For that, please see notes on [POC](../poc/README.md).

### Do you have a very broad estimate of what you're using on AWS to support the current Gen3 instance?

- We use Gen3 [Cloud Automation](https://gen3.org/resources/operator/) for production deployment and operation. Overall [architecture diagram is avail here](https://github.com/uc-cdis/cloud-automation#network-diagram).

- With Cloud Automation setup; the following are the most significant AWS resources needed to run minimally.

**AWS EC2 instances:**
- 4x Worker nodes (`t3.xlarge`)
- 1x Admin VM (`t2.micro`)
- 1x Forward Proxy VM (`t2.medium`)
 
**AWS RDS Databases:**
- 3x RDS PostgreSQL instances (`db.t2.small`)

**AWS Elasticsearch:**
- 1x Elasticsearch (`t3.small.elasticsearch`)

**AWS Elastic Kubernetes Service (EKS):**
- 1x Kubernetes cluster
 
**Others:**
- 1x Virtual Private Cloud  (VPC)
- 1x NAT Gateway
- 1x Elastic Load Balancer (ELB)
 

### How that might scale at the logic layer?
 
- Gen3 application services – [microservices](https://gen3.org/resources/developer/microservice/) – are running on Kubernetes cluster with [Horizontal Pod Autoscaler (HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/). This allows dynamically scaling of application services by monitoring Kubernetes Pods' metrics (CPU/Memory utilisation, etc).
- This then, through **_Managed_** EKS, it in-turn triggers auto-scaling of actual worker nodes/VMs. These workers are setting up to use AWS EC2 **_Auto Scaling Group_** (ASG).
- For databases and persistent layer, scaling up is possible through AWS **_Managed_** RDS and Elasticsearch services.

### Can you share cost estimation for running properly architected production-ready Gen3 instance?

- Running through aforementioned minimally scaled Cloud setup at [our pilot deployment](https://gen3.cloud.dev.umccr.org/) for past 3-4 months, we observe around **$1.5k/month** (+/-). Exclude data storage cost.

### Can you share expected skills required for deploying and operating Gen3 instance?

We'd recommend, a good decent years (minimum 3/4+) of experience on dealing with the following technology stack:

- AWS 
- Exposure to Cluster and Cloud computing
- Terraform
- Kubernetes and Docker
- Python, Bash, Linux
- ElasticSearch
- PostgreSQL (or equivalent DBA skill)
- Experience integrating Microservices application system
- Experience integrating Identity Provider (IdP), Federated AuthN/Z and Single-SignOn (SSO) setup
- Ability to debug Python/Bash code with minimum supervision
- Good troubleshooting skills
