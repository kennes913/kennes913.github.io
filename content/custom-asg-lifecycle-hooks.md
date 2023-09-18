+++
title = "node readiness with custom lifecycle hooks"
date = 2023-09-17
+++


A few months ago, **I encountered a problem**. We deployed applications as Docker containers on a large EC2 instance. These applications were networked together through Docker Compose and comprised a suite of applications for our customers. The EC2 instance used an AMI built from packer that contained all of the necessary dependencies to run the applications. For the instance to be usable, all docker containers needed to be in a healthy state otherwise we'd begin to see strange HTTP status errors. The problem began when it was requested we make these applications highly available.

To achieve high availability, we defined an auto-scaling group in Terraform. This would deploy more than one instance of the application and configure ELB health checks against a target group per Docker container, which would each have a separate path, port and response specification. The instances had already been shielded behind an edge load balancer. When a new instance was deployed within the group, the load balancer will automatically issue requests against the various target groups each of which expect a specific success response. If any of the healthchecks fail, the instance would be drained and terminated.

Given that each container exhibited varying startup times (e.g., Application A required 9 minutes, while Application B took just 3 minutes), setting up these health checks was somewhat speculative. Occasionally, we unintentionally phased out freshly deployed instances due to misconfigurations in ELB health checks. I think it's important to note that configuring the ELB healthchecks to work correctly was possible but suboptimal. You'd be constantly adjusting the checks, and any alterations to the AMI or instance necessitate reconfiguration of the health checks. For example, if you add more things to the AMI or decrease the instance size, applications would take longer to start up. To solve this issue, we decided to take control of the EC2 lifecycle.

To illustrate, here's the EC2 lifecycle during a scale-out or deployment event:  ![EC2 Lifecycle](https://docs.aws.amazon.com/images/autoscaling/ec2/userguide/images/lifecycle_hooks.png). 

Our objective became to postpone the EC2's association with the associated ELB. In essence, we aimed to deter the EC2 instance lifecycle from advancing to the 'InService' state prematurely. AWS elaborates on what happens when a node goes into the 'InService' state: _"The instance enters the InService state and the health check grace period starts. However, before the instance reaches the InService state, if the Auto Scaling group is associated with an Elastic Load Balancing load balancer, the instance is registered with the load balancer, and the load balancer starts checking its health. After the health check grace period ends, Amazon EC2 Auto Scaling begins checking the health state of the instance."_

In simpler terms, our strategy was to delay until every application (container) was fully operational and healthy prior to signaling the ASG that the instance was ready for transition to the 'InService' state. This would guarantee the successful passage of all ELB health checks and ensure that scaling events operated seamlessly without undue instance termination or oscillation.

To do this, we first added an initial lifecycle hook to the auto-scaling group in Terraform: 

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

The above ensures that the node will not automatically transition into the 'InService' state. The default action after (2000 / 60) minutes would be to automatically transition. This handles the edge case where something goes wrong with the custom lifecycle hook. 

I had to also reate a systemd job -- since we're running linux -- that would emit the node's readiness status to the ASG. Here's the systemd service file (ec2-user is root user on EC2 instances):

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

And here's the script that the service would run on start up. I've used chatGPT to generate the code below as I did not want to paste exactly what we have in source code:

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

Once all checks passed, the node would be emitted to the ASG as ready to be InService and would be deployed. This allowed us to make our applications highly available with a minimum % of nodes available at all times.