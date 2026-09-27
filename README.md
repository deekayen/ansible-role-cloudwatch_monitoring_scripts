AWS CloudWatch monitoring scripts
=========

> **Deprecated.** AWS replaced the CloudWatch Monitoring Scripts with the
> [CloudWatch agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html);
> use [deekayen.aws_cloudwatch_agent](https://github.com/deekayen/ansible-role-aws-cloudwatch-agent)
> for new hosts. This role is kept for existing EL 7/8 hosts. CI lints and
> syntax-checks it but no longer converges it on a running system.

![CI](https://github.com/deekayen/ansible-role-cloudwatch_monitoring_scripts/workflows/CI/badge.svg?branch=main) [![Project Status: Inactive – The project has reached a stable, usable state but is no longer being actively developed; support/maintenance will be provided as time allows.](https://www.repostatus.org/badges/latest/inactive.svg)](https://www.repostatus.org/#inactive) ![MIT license](https://img.shields.io/badge/license-MIT-blue)

An Ansible role to install AWS Cloudwatch Logs agent and monitoring scripts on Enterprise Linux.

Please see: https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/QuickStartEC2Instance.html

We also install the monitoring script for linux to monitor memory, cpu, swap, etc.
You need to have a role attach to the instance (or credentials) in order to be able to write to cloudwatch.

See: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/mon-scripts.html


Requirements
------------

You need to attach a role to your instance to send message to AWS Cloudwatch.

see: https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/QuickStartEC2Instance.html

Role Variables
--------------

```
aws_mon_script_install_dir: "/opt"
aws_mon_script_url: "https://aws-cloudwatch.s3.amazonaws.com/downloads/CloudWatchMonitoringScripts-1.2.2.zip"
```

Dependencies
------------

This implementation is written in perl, so some CPAN libraries are expected.

```
- role: blackstar257.perl
  vars:
    perl_cpanm_modules:
      - Switch
      - DateTime
```

This expects AWS CloudWatch Agent to be installed already, but I didn't add that as a mandatory role dependency. Here's a potential role you could run before this role in your playbook:

```
- src: git+https://github.com/riponbanik/ansible-role-aws-cloudwatch-agent.git
  name: riponbanik.aws-cloudwatch-agent
  version: 80203e57822b3c1424460a9d222f47f668e114e7
```

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: all
      roles:
        - deekayen.cloudwatch_monitoring_scripts

License
-------

MIT
