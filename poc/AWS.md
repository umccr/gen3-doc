# How to: POC setup on AWS?

- We discuss high level architecture about POC setup on AWS using Gen3 `compose-services`.
- Centre who interested to adopt Gen3 will do trial POC run before committing to a more production oriented [Cloud Automation](https://gen3.org/resources/operator/) (Kubernetes cluster) setup.
- Additionally, `compose-services` stack also gives a quick dive into Gen3 foundation services and, it is a good perk for your centre data dictionary development.

## Idea:

- We choose AWS EC2 instance `m5.2xlarge` with [Hibernation support](https://aws.amazon.com/blogs/aws/new-hibernate-your-ec2-instances/).
- Gen3 `compose-services` stack simply run on this EC2 instance.
- We hibernate this instance when not in use (over weekend, doing other priority tasks, etc).
- This EC2 instance is front-ed by ALB -- [Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html). Hence, it shows graceful `504 Gateway Time-out` when EC2 is in hibernation.
- To resume, just simply login to AWS EC2 Console and start the instance again.
- ALB is front-ed by [Route53](https://aws.amazon.com/route53/) (that's where our domain DNS created like `gen3.dev.umccr.org`) and [ACM](https://aws.amazon.com/certificate-manager/) (that's where our domain SSL cert created to work like `https://gen3.dev.umccr.org`).
- Route53 then simply has A record pointing to ALB, e.g:
  ```
  gen3.dev.umccr.org	A    gen3-compose-alb-1233456789.ap-southeast-2.elb.amazonaws.com
  ```
- Minimal running cost when EC2 instance is in hibernation = ALB + EIP + EBS

## Details:

- This idea (Route53/ACM/ALB -> EC2) is implemented in terraform stack that avail in our [infra repo](https://github.com/umccr/infrastructure/tree/master/terraform/stacks/gen3_compose).
- You can still achieve manually creating those related resources through AWS Console or CloudFormation or Ansible, etc.
- ALB network route:
  - ALB **only accept** at `:443` SSL channel. 
  - All HTTP/s traffic terminate at ALB.
  - ALB forward all traffics to EC2 instance through `:80` -- this downgrade is okay as EC2 instance is not directly reachable (i.e. private subnet + behind firewall / security group)
  - Hence, circumvent `compose-services` self-signed certificate issue
- At EC2 instance:
  - EC2 instance is inside a **Private Subnet** of given VPC.
  - Security Group (firewall) is attached to **only accept traffic coming from ALB**
  - Then, the `compose-services` stack internal Nginx `revproxy-service` take care of the rest of Layer 7 routing as is.
  - Therefore, it keeps maintain all Gen3 [microservices](https://gen3.org/resources/developer/microservice/) communication within its docker network and no exposure to its host; only except through Nginx `revproxy-service`.
  
