# Hardened Amazon Linux 2023 AMI with Packer

Automated build of a CIS-hardened Amazon Linux 2023 AMI using HashiCorp Packer. A single `packer build` produces a security-baselined machine image with Apache httpd installed, configured, and verified, with all hardening steps applied by version-controlled bash scripts and logged for audit.

## Why I built this

I spent 19 years in real-time drilling operations, where hardened, monitored, compliance-auditable systems were the job. Offshore, a system that cannot prove its own configuration state does not pass a federal audit. This project applies that same discipline to cloud infrastructure: every control is scripted, repeatable, and logged, so the image can be rebuilt identically and its security posture verified after launch.

## What it does

- Builds a custom Amazon Linux 2023 AMI from a t3.micro builder instance in us-east-2
- Applies CIS Level 1 benchmark controls via `scripts/cis-hardening.sh`, with all changes logged to `/var/log/cis-hardening.log`
- Installs and configures Apache httpd via `scripts/httpd-setup.sh`, including a custom index page listing the AMI creation date and implemented controls
- Tags the resulting AMI with project metadata and a timestamped name
- Uses a temporary security group for SSH access during the build, torn down automatically when the build completes

## Repo contents

```
.
├── template.pkr.hcl          # Packer build definition (source, build, provisioners)
├── scripts/
│   ├── cis-hardening.sh      # CIS Level 1 hardening controls
│   └── httpd-setup.sh        # Apache install, config, and index page
├── screenshots/              # Build validation and verification evidence
└── README.md
```

## CIS controls implemented

Enforced by `cis-hardening.sh`:

1. Filesystem hardening: `/tmp` mounted with `nodev`, `nosuid`, `noexec` (CIS 1.1.1.1)
2. SSH configuration: root login disabled, protocol 2 enforced, empty passwords disabled (CIS 5.2.8)
3. Password aging and complexity policies (CIS 5.4.1.1, 5.4.1.2)
4. auditd enabled with rules monitoring sensitive files (CIS 4.1.1.1)
5. Network hardening: IPv6 disabled, ICMP redirects disabled, SYN cookies enabled (CIS 3.3.1, 3.3.2)
6. File permissions locked down on `/etc/passwd`, `/etc/shadow`, `/etc/group`
7. Login banner set in `/etc/issue.net` with a compliance notice

## Apache configuration

`httpd-setup.sh` installs Apache httpd and mod_ssl, configures the service to start on boot, applies basic hardening (`ServerTokens`, `ServerSignature`), and writes a custom `index.html` displaying the AMI creation date and the list of implemented CIS controls, so a launched instance self-documents its baseline.

## Building the AMI

Prerequisites: an AWS account with EC2 and IAM permissions, AWS CLI configured with credentials, and Packer v1.14.2 or later.

```bash
packer init .
packer validate template.pkr.hcl
packer build template.pkr.hcl
```

The build launches a temporary EC2 instance, runs both provisioning scripts over SSH, creates the AMI, and terminates the builder.

![Template validated](screenshots/Packer_Validated.png)

![AMI build complete](screenshots/AMI_Build_Complete_YIPPEE.png)

## Verifying a launched instance

Launch an EC2 instance from the AMI, then:

```bash
# 1. SSH in
ssh -i ~/your-key.pem ec2-user@<instance-public-ip>

# 2. Confirm CIS controls took effect
sudo grep -E "PermitRootLogin|Protocol|PermitEmptyPasswords" /etc/ssh/sshd_config
grep -E "PASS_MAX_DAYS|PASS_MIN_DAYS" /etc/login.defs
sudo systemctl status auditd

# 3. Confirm Apache is serving the hardened index page
curl http://localhost
```

![Custom httpd page](screenshots/Custom_HTTPD_Page.png)

## Challenges and solutions

**1. Packer install blocked by a conflicting apt source.** Installing Packer from the HashiCorp apt repository failed on my Ubuntu machine because a stale Helm repository entry (`baltocdn.com`) in `/etc/apt/sources.list.d/` conflicted during `apt update`. Diagnosed it with `grep -r "baltocdn.com" /etc/apt/sources.list.d/`, removed the stale list file, and the install completed cleanly. Lesson: when a package manager misbehaves, audit the source lists before blaming the package.

**2. Git authentication failing over HTTPS.** Pushing to GitHub with username and password failed because GitHub removed password authentication for Git operations in 2021. Generated an ed25519 SSH key pair, added the public key to my GitHub account, switched the remote to SSH, and verified with `ssh -T git@github.com`.

**3. HCL validation errors on the security group CIDR.** Early builds failed `packer validate` on the temporary security group configuration. The fix was correcting the `temporary_security_group_source_cidrs` argument to pass the CIDR as a list (`["0.0.0.0/0"]`) rather than a bare string. Running `packer validate` before every build caught this and several similar formatting issues before they cost a build cycle.

## Cleanup

To avoid ongoing AWS charges after testing:

1. Deregister the AMI: EC2 > AMIs > select the AMI > Actions > Deregister
2. Delete the associated snapshot: EC2 > Snapshots > select the snapshot linked to the AMI > Actions > Delete Snapshot

---

*Built as a project for a cloud engineering course, extended and documented independently.*
