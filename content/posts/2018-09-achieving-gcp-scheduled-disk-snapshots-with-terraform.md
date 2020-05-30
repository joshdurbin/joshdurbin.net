+++
title = "Achieving GCP scheduled disk snapshots with Terraform"
description = "Scheduled disk snapshots announced at NEXT 2018 achievable with gcloud cli and terraform local-exec provisioners"
date = "2018-09-25"
tags = ["gcloud", "terraform" ]
+++

Google announced scheduled disk snapshots at Google Cloud NEXT this year. The support
allows admins/operators to create schedules and bind disks to the schedules. Backed by some
sort of quartz/timer trigger, the schedules initiate snapshots of disks and maintain copies for
a configurable period of time.

The details for these alpha features can be difficult to dig up on the web, but are pretty easy to uncover
via gclouds `alpha` and `beta` components. Upon first execution of any such command, the CLI will 
request authorization for installation. This can also explicitly be done by using the `gcloud components install [alpha|beta]`
command.

The following image shows the basics of creating and attaching a scheduled snapshot:

![image](/img/2018-09-achieving-gcp-scheduled-disk-snapshots-with-terraform/1.png)

Additional usage details are available from the gcloud CLI by suffixing any given command with `help` i.e.
`gcloud alpha compute resource-policies create-back-schedule help`. Here are a few more examples of 
schedule creation:

![image](/img/2018-09-achieving-gcp-scheduled-disk-snapshots-with-terraform/2.png)

Unfortunately, the [Terraform GCP Provider](https://github.com/terraform-providers/terraform-provider-google) does not currently support beta or alpha features of GCP, or, at least
not this one. However, this lack of support doesn't mean the feature can't be leveraged.
Terraform's [`local-exec`](https://www.terraform.io/docs/provisioners/local-exec.html) provisioners allow Terraform to stitch resources and lifecycle
events into any local command line tool. The only requirement for any external call
from terraform, locally, is that the environment
is properly setup to make use of a given command. Consider the following example:

Given a pre-existing policy `snap12keep168` created by a command like:

`gcloud alpha compute resource-policies create-backup-schedule snap12keep168 --description "snapshots every 12 keeps for 168" --start-time 00:00 --hourly-schedule=12 --max-retention-days=7 --region us-west1a`

...and a Terraform HCL code snippet of:

```terraform
variable "snapshot_policy_name" {
  default = "snap12keep168"
}

data "google_project" "project" {}

resource "google_compute_disk" "important_data" {
  zone  = "us-west1a"
  name  = "important_data"
  size  = "2000"

  provisioner "local-exec" {
    command = "if [[ -n '${var.snapshot_policy_name}' ]]; then gcloud alpha compute disks add-resource-policies ${self.name} --resource-policies ${var.snapshot_policy_name} --zone ${self.zone} --project ${data.google_project.project.id}; fi"
  }

  provisioner "local-exec" {
    when    = "destroy"
    command = "if [[ -n '${var.snapshot_policy_name}' ]]; then gcloud alpha compute disks remove-resource-policies ${self.name} --resource-policies ${var.snapshot_policy_name} --zone ${self.zone} --project ${data.google_project.project.id}; fi"
  }
} 
```

The first `local-exec` block fires after the resource is created executing a shell command
only if the variable `snapshot_policy_name` is populated. If it is, it will call out to Google
via the gcloud CLI using the alpha feature support and bind the disk to the policy. The second `local-excec`
block fires before the resource is destroyed ensuring proper cleanup.

In the event that gcloud alpha support has not been enabled; the command will fail and Terraform will
halt. If the exit code is non-zero for the resource creation provisioner, sub-sequent runs will taint the previously
created resource and try again.

Though this feature is available in the gcloud CLI, it is not enabled on projects by default
and requires writing to Google Cloud support to get enabled.

Happy snapshotting! 
 