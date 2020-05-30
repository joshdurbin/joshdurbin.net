+++
title = "ASG Rolling Updates on AWS with Terraform"
date = "2018-05-01"
tags = ["aws", "terraform"]
aliases = [ "/blurbs/2018-05-auto-scaling-rollout-on-aws-with-terraform/"]
+++

The cloud is great. Isn't the cloud great? ...and because you're reading this you know
about running compute instances in the cloud. If you're unlucky enough to still be dealing
with code deployed to machines instead of containers, you might be in the painful world of
shipping code by automated or manual move and re-start procedures. Perhaps you are lucky enough
to be in a containerized environment, but not on Kubernetes. Maybe you're using [Nomad](https://www.nomadproject.io) to manage
your containers and thus you're still producing custom machine images etc...
If so, you're aware of the concept of machines, machine groups, and load balancers. If not, a quick primer:

Load Balancers direct requests from a "single point" to an internal pool of resources, [Auto Scaling Groups (ASG)](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html) (or [Managed Instance Groups](https://cloud.google.com/compute/docs/instance-groups/updating-managed-instance-groups) in GCP).
Load Balancers are typically exposed to the outside world and often used for SSL termination, bridging traffic
from the outside world to an internal private network. In addition to the Load Balancers
there are the network definitions / rules, ASG definitions, machine images (on AWS known as [AMIs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html)), and
[Launch Configurations](https://docs.aws.amazon.com/autoscaling/ec2/userguide/LaunchConfiguration.html) which bridge machine images and the type of machine to the ASG.

One great feature of these group is the ability to roll image updates across the "fleet".
Unlike Googles Managed Instance Groups, AWS ASG resources do not
natively support rolling update policies. The rolling update policy, monitoring,
and execution require life-cycle awareness within the platform to enact changes on ASGs.
The most basic example of this is rolling new AMIs via Launch Configuration changes across
a managed set of machines. This requires that the ASG be provisioned via a [Cloudformation template](https://aws.amazon.com/cloudformation/aws-cloudformation-templates/).
[Cloudformation](https://aws.amazon.com/cloudformation/) is the conduit allowing for the attachment to resources and monitoring
of qualifying life-cycle events and prescribed actions. [Terraform](https://www.terraform.io), [Ansible](https://www.ansible.com), and
other infrastructure provisioning tools, or even more actively managed tools such as [Chef](https://www.chef.io/chef/)
and [Puppet](https://puppet.com) lack the life-cycle awareness necessary to react.

That said, achieving rolling updates within an ASG with Terraform is easily
with a few additional resources over a traditional ASG.

First let's review a very basic example of an ASG without monitoring, auto scaling,
or rollout policy. **Note**: These examples reference a statically defined [VPC](https://aws.amazon.com/vpc/), [AV Zones](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html), and [ELB](https://aws.amazon.com/elasticloadbalancing/).

## Basic ASG Monitoring and Scaling

For the most basic example we have the following resources and data in Terraform:

- Data querying for the most recent Ubuntu AMI
- A launch config (without showing all the networking resources)
- An ASG

```terraform
data "aws_ami" "most_recent_ubuntu_xenial" {

  most_recent = true

  filter {
    name = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-xenial-16.04-amd64-server-*"]
  }

  filter {
    name = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["099720109477"] # Canonical
}

resource "aws_launch_configuration" "sample_lc" {

  image_id = "${data.aws_ami.most_recent_ubuntu_xenial.id}"
  instance_type = "t2.medium"
}

resource "aws_autoscaling_group" "sample_asg" {

    availability_zones = ["us-west-2a", "us-west-2b"]
    load_balancers = ["elb-123456"]
    vpc_zone_identifier = ["vpc-abc1234"]
    launch_configuration = "${aws_launch_configuration.sample_lc.name}"
    max_size = 5
    min_size = 2
}
```

Note that the min, max size of the ASG are defined, but have no
monitoring or event triggers to scale up from 2 or down from 5, etc... Manipulating
the current count in this case requires manual action, typically
API calls from custom code.

### Monitoring

There are [many different metrics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CW_Support_For_AWS.html) available
for scrutiny within the ASG resources. The one we are going to focus on is:

- `CPUUtilization`

The following block defines a [Cloudwatch Metric Alarm](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html) that will average the CPU utilization
for the machines in an ASG over a 60 second window and trigger an alarm state when that average
is >= the treshhold of `75`. This functions as our upper bound...

```terraform
resource "aws_cloudwatch_metric_alarm" "scaling_app_high" {

    alarm_name = "sample-cpu-utilization-exceeds-normal"
    comparison_operator = "GreaterThanOrEqualToThreshold"
    evaluation_periods = "1"
    metric_name = "CPUUtilization"
    namespace = "AWS/EC2"
    period = "60"
    statistic = "Average"
    threshold = "75"

    dimensions {
        AutoScalingGroupName = "${aws_autoscaling_group.sample_asg.name}"
    }
}
```

The next block is similar, but with different comparison operators and thresholds. It also
averages over a longer period of time because scale in events should only occur after whatever
event triggering scale out due to load is over.

```terraform
resource "aws_cloudwatch_metric_alarm" "scaling_app_low" {

    alarm_name = "sample-cpu-utilization-normal"
    comparison_operator = "LessThanThreshold" #  <------- CHANGED
    evaluation_periods = "1"
    metric_name = "CPUUtilization"
    namespace = "AWS/EC2"
    period = "300" #  <---------------------------------- CHANGED
    statistic = "Average"
    threshold = "60" #  <-------------------------------- CHANGED

    dimensions {
        AutoScalingGroupName = "${aws_autoscaling_group.sample_asg.name}"
    }
}
```

These Cloudwatch Metric Alarm resources in their current state are only alarms
and are in capable of influencing scale events on the ASG. In order for the Cloudwatch
Metric Alarms to affect the ASG we have to define [Autoscaling Policies](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-simple-step.html) and bind the
alarms to said policies.

### Scaling Policies

The scaling policies are the magic bridge between our aforecreated alarms
and the changes to our ASGs (within the ASG pre-defined lower and upper bounds).

The following two policies achieve scale up by one and down by one via the `ChangeInCapacity` adjustment type:

```terraform
resource "aws_autoscaling_policy" "scale_out_scaling_app" {

    name = "scale-out-cpu-high"
    scaling_adjustment = 1
    adjustment_type = "ChangeInCapacity"
    cooldown = 300
    autoscaling_group_name = "${aws_autoscaling_group.sample_asg.name}"
}

resource "aws_autoscaling_policy" "scale_in_scaling_app" {

    name = "scale-in-cpu-normal"
    scaling_adjustment = -1
    adjustment_type = "ChangeInCapacity"
    cooldown = 300
    autoscaling_group_name = "${aws_autoscaling_group.sample_asg.name}"
}
```

### Tying the knot

Now that we have our Auto Scaling policies defined, we need to alter our Cloudwatch Metric
Alarms to invoke these policies when an alarm condition is raised. Achieving this union
is simple with the definition of the `alarm_action` array of [ARNs](https://docs.aws.amazon.com/general/latest/gr/aws-arns-and-namespaces.html) on the target
`aws_cloudwatch_metric_alarm` resources.

Making these changes yields updated Cloudwatch Metric Alarms...:

```terraform
resource "aws_cloudwatch_metric_alarm" "scaling_app_high" {

    alarm_name = "sample-cpu-utilization-exceeds-normal"
    comparison_operator = "GreaterThanOrEqualToThreshold"
    evaluation_periods = "1"
    metric_name = "CPUUtilization"
    namespace = "AWS/EC2"
    period = "60"
    statistic = "Average"
    threshold = "75"

    dimensions {
        AutoScalingGroupName = "${aws_autoscaling_group.sample_asg.name}"
    }

    alarm_actions = ["${aws_autoscaling_policy.scale_out_scaling_app.arn}"]
}

resource "aws_cloudwatch_metric_alarm" "scaling_app_low" {

    alarm_name = "sample-cpu-utilization-normal"
    comparison_operator = "LessThanThreshold"
    evaluation_periods = "1"
    metric_name = "CPUUtilization"
    namespace = "AWS/EC2"
    period = "300"
    statistic = "Average"
    threshold = "60"

    dimensions {
        AutoScalingGroupName = "${aws_autoscaling_group.sample_asg.name}"
    }

    alarm_actions = ["${aws_autoscaling_policy.scale_out_scaling_app.arn}"]
}
```

## Basic ASG rollout policy support

As previously mentioned in the beginning of this post, rollout policy support
requires the definition of *something* that's aware of life-cycle events happening to
resources within AWS (like the Launch Config). Terraform, Ansible, and other infrastructure
tools that run to completion and cloud provider/machine states match the prescribed
"code" do not support this.

In order to achieve this, we have to drop the definition we presently have of our
`aws_autoscaling_group` resource `sample_asg` and replace it with a Cloudformation Template
that does the same thing (with the addition of the [rollout policy](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-updatepolicy.html).

The Cloudformation template creates the AWS resources (much like Terraform) and has the
advantage of being aware of life-cycle events within the AWS ecosystem.

### Cloudformation Template

Our template will place particular emphasis on allowing for changes to the following aspects
of the ASG and the update policy:

- Availability Zone
- Launch Config
- VPC Zone ID
- Load Balancer Name
- Min and max capacity
- The rollout update pause time

Our terraform AWS Cloudformation Stack resource defines:

1. The name of the stack
2. The parameters passed to the stack
3. The stack itself

Within the Cloudformation stack or template, there are four sections (defined as the `template_body`
within the opening and closing multiline `STACK` keyword):

1. Description (self explanatory)
2. Parameters
3. Resources defined
4. Outputs

```terraform
resource "aws_cloudformation_stack" "rolling_update_asg" {

    name = "asg-rolling-update"
    template_body = "${file("${path.module}/cloudformation-asg.json")}"

    parameters {

        AvailabilityZones = "us-west-2a,us-west-2b"
        LaunchConfigurationName = "${aws_launch_configuration.sample_lc.name}"
        VPCZoneIdentifier = "vpc-abc1234"
        LoadBalancerNames = "elb-123456"
        MaximumCapacity = "5"
        MinimumCapacity = "2"
        UpdatePauseTime = "60"
    }

      template_body = <<STACK
{
  "Description": "ASG cloud formation template",
  "Parameters": {
    "AvailabilityZones": {
      "Type": "CommaDelimitedList",
      "Description": "The availability zones to be used for the app"
    },
    "LaunchConfigurationName": {
      "Type": "String",
      "Description": "The launch configuration name"
    },
    "VPCZoneIdentifier": {
      "Type": "List<AWS::EC2::Subnet::Id>",
      "Description": "The VPC subnet IDs"
    },
    "LoadBalancerNames": {
      "Type": "CommaDelimitedList",
      "Description": "The load balancer names for the ASG"
    },
    "MaximumCapacity": {
      "Type": "String",
      "Description": "The maximum desired capacity size"
    },
    "MinimumCapacity": {
      "Type": "String",
      "Description": "The minimum and initial desired capacity size"
    },
    "UpdatePauseTime": {
      "Type": "String",
      "Description": "The pause time during rollout for the application"
    }
  },
  "Resources": {
    "ASG": {
      "Type": "AWS::AutoScaling::AutoScalingGroup",
      "Properties": {
        "AvailabilityZones": { "Ref": "AvailabilityZones" },
        "LaunchConfigurationName": { "Ref": "LaunchConfigurationName" },
        "MaxSize": { "Ref": "MaximumCapacity" },
        "MinSize": { "Ref": "MinimumCapacity" },
        "DesiredCapacity": { "Ref": "MinimumCapacity" },
        "VPCZoneIdentifier": { "Ref": "VPCZoneIdentifier" },
        "LoadBalancerNames": { "Ref": "LoadBalancerNames" },
        "TerminationPolicies": [ "OldestLaunchConfiguration", "OldestInstance" ],
        "HealthCheckType": "ELB",
        "MetricsCollection": [
          {
            "Granularity": "1Minute",
            "Metrics": [ ]
          }
        ],
        "HealthCheckGracePeriod": "300",
        "Tags": [ ]
      },
      "UpdatePolicy": {
        "AutoScalingRollingUpdate": {
          "MinInstancesInService": "2",
          "MaxBatchSize": "1",
          "PauseTime": { "Ref": "UpdatePauseTime" }
        }
      }
    }
  },
  "Outputs": {
    "AsgName": {
      "Description": "ASG reference ID",
      "Value": { "Ref": "ASG" }
    }
  }
}
STACK
}
```

Terraform passes values to the Cloudformation template via the parameters section
of template and receives values via the outputs.

The `UpdatePolicy` sub-document in the JSON template is the magic sauce that controls
updates. Read more about it [here](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-updatepolicy.html).
Our `UpdatePolicy` hard codes the min instances in service and max batch size and allow
for dynamic configurations of the update `PauseTime` (for launch configs that spin up faster/slower than others and pass health checks, in this case).

```
"UpdatePolicy": {
  "AutoScalingRollingUpdate": {
    "MinInstancesInService": "2",
    "MaxBatchSize": "1",
    "PauseTime": {
      "Ref": "UpdatePauseTime"
    }
  }
}
```

Note that the `PauseTime` (and other Cloudformation variables) use a JSON sub-document
that references a particular key and that key machines a parameter definition in the Cloudformation
template... which matches a parameter passed from Terraform.

Finally, take note of the `Outputs` section of the Cloduformation template. The outputs are
made available to the Terraform resource and are referenceable. In this case, we expose
the name of the ASG as `AsgName` and can access this value in Terraform via:
`${aws_cloudformation_stack.rolling_update_asg.outputs["AsgName"]}`.

### Integration with Metric, alarm, scaling policy resources

The final step in this example is to link our previously created metric/alarm and scaling policy
resources with the ASG defined in our Cloudformation template. (Note: that we don't have to link
the Launch Configuration because it's already passed to the Cloudformation template itself.)

Doing so essentially involves changing the `autoscaling_group_name` attribute on the `aws_autoscaling_policy` resource
using the Cloudformation template output we just discussed.:

```terraform
resource "aws_autoscaling_policy" "scale_out_scaling_app" {

    name = "scale-out-cpu-high"
    scaling_adjustment = 1
    adjustment_type = "ChangeInCapacity"
    cooldown = 300
    autoscaling_group_name = "${aws_cloudformation_stack.rolling_update_asg.outputs["AsgName"]}"
}

resource "aws_autoscaling_policy" "scale_in_scaling_app" {

    name = "scale-in-cpu-normal"
    scaling_adjustment = -1
    adjustment_type = "ChangeInCapacity"
    cooldown = 300
    autoscaling_group_name = "${aws_cloudformation_stack.rolling_update_asg.outputs["AsgName"]}"
}
```

...and changing the `AutoScalingGroupName` dimension on the `aws_cloudwatch_metric_alarm` resource:

```terraform
resource "aws_cloudwatch_metric_alarm" "scaling_app_high" {

    alarm_name = "sample-cpu-utilization-exceeds-normal"
    comparison_operator = "GreaterThanOrEqualToThreshold"
    evaluation_periods = "1"
    metric_name = "CPUUtilization"
    namespace = "AWS/EC2"
    period = "60"
    statistic = "Average"
    threshold = "75"

    dimensions {
        AutoScalingGroupName = "${aws_cloudformation_stack.rolling_update_asg.outputs["AsgName"]}"
    }

    alarm_actions = ["${aws_autoscaling_policy.scale_out_scaling_app.arn}"]
}

resource "aws_cloudwatch_metric_alarm" "scaling_app_low" {

    alarm_name = "sample-cpu-utilization-normal"
    comparison_operator = "LessThanThreshold"
    evaluation_periods = "1"
    metric_name = "CPUUtilization"
    namespace = "AWS/EC2"
    period = "300"
    statistic = "Average"
    threshold = "60"

    dimensions {
        AutoScalingGroupName = "${aws_cloudformation_stack.rolling_update_asg.outputs["AsgName"]}"
    }

    alarm_actions = ["${aws_autoscaling_policy.scale_out_scaling_app.arn}"]
}
```

### Tada, magic

...and there you have it! Any changes to Launch Configuration will change the input
to the Cloudformation template that creates your ASG and the remaining infrastructure
is directly managed by Terraform.

If, at this point, you're asking yourself why you wouldn't just do all of this in Cloudformation,
my answer for you would be... Cloudformatio is key, again, to resources that require life cycle
awareness. Other than this, though, you're dealing with a syntax that's more difficult to work
with (JSON over HCL), and vendor lock in. Terraform shines when you link multiple cloud
providers together, leveraging each for the best tasks its capable of (Kubernetes on GCP for example).
Also, [here is](https://www.terraform.io/intro/vs/cloudformation.html) Hashicorp's response to that question.
