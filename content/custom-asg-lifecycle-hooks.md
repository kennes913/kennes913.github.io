+++
title = "node readiness with custom lifecycle hooks"
date = 2023-09-17
+++


**I faced a problem a few months ago.** We deployed applications as docker containers on one large EC2 instance. These applications were networked together and comprised a suite of applications that we were offering to customers. The EC2 instance used an AMI built from Packer that contained all of the necessary dependencies to run the applications. For the instance to be usable, all docker containers needed to be in a healthy state otherwise we'd begin to see strange HTTP status errors. The problem began when it was requested we make these applications highly available.

To get to a place where high availability was possible, we defined an auto-scaling group in Terraform that would deploy > 1 instances of the application and set up ELB health checks against a target group containing these instances (the instances we're already exposed behind and edge load balancer). We had a target group per application, each with different path, port and response specifications. Once an instance goes into service, the load balancer will automatically issue requests against the target group on a specific path and port. It will expect a specific HTTP status code (e.g. 200) as the response.  In this case, the ELB was issuing healthchecks against multiple target groups which contained the _same instances_. If any of the ELB healthchecks fail, the node gets drained and terminated.

Since each container had varied start up times (e.g. application A takes 9 minutes to start up; application B takes 3 minutes), configuring these healthchecks became quite a bit of guesswork and at times we would incorrectly cycle out newly deployed instances due to ELB healthcheck configuration issues. It's important to note that this could be done, but it's not optimal because you're finessing how the checks would work and changes to the AMI or the instance itself would force you to reconfigure the healthchecks. In order to alleviate this problem, we decided to take control of the EC2 lifecycle. 

You can see the EC2 lifecycle during a scale out / deployment event: ![EC2 Lifecycle](https://docs.aws.amazon.com/images/autoscaling/ec2/userguide/images/lifecycle_hooks.png). 

What we wanted to do was to prevent the EC2 from being registered with the assocaited ELB prematurely. This means preventing the EC2 instance lifecycle from reaching an InService state. AWS talks about that here, "_The instance enters the InService state and the health check grace period starts. However, before the instance reaches the InService state, if the Auto Scaling group is associated with an Elastic Load Balancing load balancer, the instance is registered with the load balancer, and the load balancer starts checking its health. After the health check grace period ends, Amazon EC2 Auto Scaling begins checking the health state of the instance._".

To clarify, the idea would be to wait until all applications (containers) are up and healthy before telling the ASG that the instance is ready to be moved into the InService state.This would ensure that all ELB healthchecks would pass and scaling events would work correctly without instance termination flapping.


First we added an initial lifecycle hook to the autoscaling group in Terraform: 

```hcl
... 
  resource "aws_autoscaling_group" "bar" {
    ...
    initial_lifecycle_hook {
      name                 = "init_lifecycle_hook"
      default_result       = "CONTINUE"
      heartbeat_timeout    = 2000
      lifecycle_transition = "autoscaling:EC2_INSTANCE_LAUNCHING"
    }
    ...
  }
...
```

Without the above, the instance would automatically proceed to an InService state. The above ensures that it stays in that Pending:Wait state from the diagram above for > 30 minutes.

Since we're working with EC2 instances which typically are running linux, I had to create a systemd job that would emit the node's readiness status to the ASG. First, I added the systemd service file:

```toml
[Unit]
Description=Application readiness checks.

[Service]
Type=simple
ExecStart=/script/dir/file.sh
Restart=on-failure
WorkingDirectory=/script/dir/
RestartSec=30
User=ec2-user

[Install]
WantedBy=default.target
```

Here's the script that the service would run on start up. I've used chatGPT to generate the code below as I did not want to paste exactly what we have in source code:

```bash
# /script/dir/file.sh

main() {

  echo "Get EC2 instance information"

  TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`
  INSTANCE_ID=`curl -H "X-aws-ec2-metadata-token: $TOKEN" -v http://169.254.169.254/latest/meta-data/instance-id`
  REGION=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" -v http://169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region)

  echo "Get Auto Scaling Group information"

  ASG_NAME=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID --region $REGION --query "Reservations[*].Instances[*].Tags[?Key=='aws:autoscaling:groupName'].Value" --output text)
  LIFECYCLE_HOOKS=$(aws autoscaling describe-lifecycle-hooks --auto-scaling-group-name $ASG_NAME --region $REGION)
  LIFECYCLE_HOOK_NAME=$(echo $LIFECYCLE_HOOKS | jq -r '.LifecycleHooks[] .LifecycleHookName')

  while true; do
    # Your custom application checks go here. Once all pass, exit loop.
  done

  aws autoscaling complete-lifecycle-action \
    --region "${REGION}" \
    --auto-scaling-group-name "${ASG_NAME}" \
    --instance-id "${INSTANCE_ID}" \
    --lifecycle-hook-name "${LIFECYCLE_HOOK_NAME}" \
    --lifecycle-action-result CONTINUE

  echo "Instance emitted to auto scaling group (${ASG_NAME})."

}
```

Once all checks passed, the node would be emitted to the ASG as ready to be InService and would be deployed. This allowed us to make our applications highly available with a minimum % of nodes in the ASG available at all times.