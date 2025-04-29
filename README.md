## EC2 Maintenance Automation

This automation runs maintenance tasks on EC2 instances and handles their registration with a load balancer. Instances are temporarily removed from the target group during maintenance and re-added once completed to avoid serving traffic during updates.
