# deekayen.cloudwatch_monitoring_scripts

[![CI](https://github.com/deekayen/ansible-role-cloudwatch_monitoring_scripts/actions/workflows/ci.yml/badge.svg)](https://github.com/deekayen/ansible-role-cloudwatch_monitoring_scripts/actions/workflows/ci.yml) [![Ansible Galaxy](https://img.shields.io/badge/galaxy-deekayen.cloudwatch__monitoring__scripts-blue.svg)](https://galaxy.ansible.com/ui/standalone/roles/deekayen/cloudwatch_monitoring_scripts/) [![Project Status: Inactive – The project has reached a stable, usable state but is no longer being actively developed; support/maintenance will be provided as time allows.](https://www.repostatus.org/badges/latest/inactive.svg)](https://www.repostatus.org/#inactive) ![MIT license](https://img.shields.io/badge/license-MIT-blue)

> **Deprecated.** AWS replaced the CloudWatch Monitoring Scripts with the
> [CloudWatch agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html);
> use [deekayen.aws_cloudwatch_agent](https://github.com/deekayen/ansible-role-aws-cloudwatch-agent)
> for new hosts. This role is kept for existing EL 7/8 hosts. CI lints and
> syntax-checks it but no longer converges it on a running system.

An Ansible role that installs the AWS CloudWatch Monitoring Scripts, the Perl `mon-put-instance-data.pl` tool, on CentOS and RHEL 7 and 8, and schedules it to publish memory, swap, and root disk metrics to CloudWatch every five minutes.

The role installs the Perl modules and packages the scripts need, downloads `CloudWatchMonitoringScripts-1.2.2.zip` from `aws-cloudwatch.s3.amazonaws.com` and unpacks it into `/opt/aws-scripts-mon`, and writes a root cron job to `/etc/cron.d/ansible_aws_mon-put-instance-data`.

## Requirements

- ansible-core 2.15 or newer on the controller. EL 7 targets need ansible-core 2.16 or older; as of October 2026, the [Ansible support matrix](https://docs.ansible.com/ansible/latest/reference_appendices/release_and_maintenance.html) says Python 2.7 target support ends with 2.16.
- The `community.general` collection, which the `blackstar257.perl` dependency uses for `cpanm`.
- Outbound HTTPS from the target to `aws-cloudwatch.s3.amazonaws.com`, plus access to CPAN for the Perl modules.
- Privilege escalation on the target. Run the play with `become: true`; the role installs packages, writes under `/opt`, and adds a root cron job.
- AWS credentials or an EC2 instance role that can publish CloudWatch metrics, available to root's cron job.
- Fact gathering left on. The package tasks and the second dependency branch on `ansible_facts.distribution` and `ansible_facts.distribution_major_version`.

## Supported platforms

| Platform | Versions |
| --- | --- |
| EL (CentOS, RHEL) | 7, 8 |

CI runs `ansible-lint` and `ansible-playbook --syntax-check` only.

## Installation

From Ansible Galaxy, which also installs the `blackstar257.perl` dependency:

```bash
ansible-galaxy role install deekayen.cloudwatch_monitoring_scripts
ansible-galaxy collection install community.general
```

Or pin it in `requirements.yml`:

```yaml
---
roles:
  - name: deekayen.cloudwatch_monitoring_scripts
    src: https://github.com/deekayen/ansible-role-cloudwatch_monitoring_scripts.git
    scm: git
    version: main

collections:
  - name: community.general
```

```bash
ansible-galaxy install -r requirements.yml
```

## Role variables

| Variable | Default | Description |
| --- | --- | --- |
| `aws_mon_script_install_dir` | `/opt` | Directory the zip is unpacked into. The scripts land in `<dir>/aws-scripts-mon`, and the cron job runs them from there. Must be an absolute path; the role asserts this. |
| `aws_mon_script_url` | `https://aws-cloudwatch.s3.amazonaws.com/downloads/CloudWatchMonitoringScripts-1.2.2.zip` | Source zip. Must be an `http://` or `https://` URL ending in `.zip`; the role asserts this. Point it at an internal mirror if the host cannot reach S3. |

`vars/main.yml` lists `crontabs`, `zip`, and `unzip` as package dependencies; it is an internal value.

## Behavior

- The cron job runs `mon-put-instance-data.pl --mem-used --swap-util --mem-util --mem-used --mem-avail --disk-space-util --disk-path=/ --from-cron` every five minutes as root. Only `/` is reported for disk space.
- On EL 7, the role installs `perl-Sys-Syslog` and `perl-LWP-Protocol-https` from packages. On EL 8, the second `blackstar257.perl` dependency installs `Sys::Syslog` and `LWP::Protocol::https` from CPAN instead.

## Dependencies

Both entries in `meta/main.yml` use [blackstar257.perl](https://galaxy.ansible.com/ui/standalone/roles/blackstar257/perl/):

- On every host, it installs `Switch`, `DateTime`, and `Digest::SHA` with `cpanm`.
- On CentOS and RedHat 8 only, it installs `Sys::Syslog` and `LWP::Protocol::https`.

## Example playbook

```yaml
---
- name: Publish memory and disk metrics from legacy EL hosts.
  hosts: el7_legacy
  become: true

  vars:
    aws_mon_script_url: https://mirror.example.internal/aws/CloudWatchMonitoringScripts-1.2.2.zip

  roles:
    - deekayen.cloudwatch_monitoring_scripts
```

`mirror.example.internal` is a placeholder for an internal mirror.

## Known issues

- The package tasks (`tasks/main.yml:26` and `:35`) and the EL 8 dependency (`meta/main.yml:42`) run only when `ansible_facts.distribution` is `CentOS` or `RedHat`. On Rocky Linux, AlmaLinux, or Oracle Linux, the role skips installing `unzip`, which the `unarchive` task needs, and the EL 8 Perl modules.
- The cron job passes `--mem-used` twice (`tasks/main.yml:57-58`).

## Development

CI runs on every push to `main` and every pull request (see `.github/workflows/ci.yml`). It installs `blackstar257.perl` and `community.general` from `tests/requirements.yml`, runs `ansible-lint --profile production`, and syntax-checks `tests/test.yml`. To run the same checks locally:

```bash
pip3 install ansible-lint
ansible-galaxy install -r tests/requirements.yml
ansible-lint --profile production
mkdir -p .ansible/roles && ln -sfn "$PWD" .ansible/roles/deekayen.cloudwatch_monitoring_scripts
ANSIBLE_ROLES_PATH=.ansible/roles:~/.ansible/roles ansible-playbook --syntax-check tests/test.yml -i tests/inventory
pre-commit run --all-files
```

`.pre-commit-config.yaml` runs `check-yaml`, `end-of-file-fixer`, `trailing-whitespace`, `yamllint`, `flake8`, and `ansible-lint --profile production`.

### Repository layout

| Path | Purpose |
| --- | --- |
| `tasks/main.yml` | Package installs, script download, and cron job. |
| `tasks/assert.yml` | Input validation, tagged `always`. |
| `defaults/main.yml` | Every user-facing variable. |
| `vars/main.yml` | Package dependency list. |
| `meta/main.yml` | Galaxy metadata, platforms, and the `blackstar257.perl` dependencies. |
| `tests/` | Syntax-check playbook, inventory, and test requirements used by CI. |
| `.github/workflows/` | `ci.yml` for lint and syntax check, `release.yml` for Galaxy import. |

## Releases

Pushing a git tag runs `.github/workflows/release.yml`, which imports the tagged commit into Ansible Galaxy as `deekayen.cloudwatch_monitoring_scripts`. The import needs a `GALAXY_API_KEY` repository or organization secret.

## License

MIT. See [LICENSE](LICENSE).

## Authors

The LICENSE copyright is held by Mijndert Stuij (2016). This repository is forked from [bitintheskud/ansible-role-cloudwatch-logs-agent](https://github.com/bitintheskud/ansible-role-cloudwatch-logs-agent), itself a fork of [chaordic/ansible-role-cloudwatch-logs-agent](https://github.com/chaordic/ansible-role-cloudwatch-logs-agent), and maintained by [David Norman](https://github.com/deekayen). Sponsorship links are in [.github/FUNDING.yml](.github/FUNDING.yml).
