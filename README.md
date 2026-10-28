# terraform-project-execs

Terraform project that provisions an Ubuntu EC2 instance in AWS (`ap-south-1`) inside its own VPC, then uses **provisioners** to copy a small Flask app onto the instance and start it.

## What it creates

| Resource | Purpose |
| --- | --- |
| `aws_vpc.myvpc` | VPC with CIDR `10.0.0.0/16` |
| `aws_subnet.sub1` | Public subnet `10.0.0.0/24` in `ap-south-1a` |
| `aws_internet_gateway.igw` | Internet access for the VPC |
| `aws_route_table.RT` + `rta1` | Routes `0.0.0.0/0` through the IGW |
| `aws_security_group.webSg` | Allows inbound HTTP (80) and SSH (22) |
| `aws_key_pair.example` | Uploads your local `~/.ssh/id_rsa.pub` as `terraform-noman` |
| `aws_instance.server` | `t3.micro`, Ubuntu 22.04 (`ami-082a1ff88c78c6290`), tagged `Name = terraform-flask-server` |

On the instance, Terraform:

1. **`file` provisioner** – copies `app.py` to `/home/ubuntu/app.py`
2. **`remote-exec` provisioner** – installs `python3-pip` and Flask, then starts the app on port 80 in the background with `nohup` (output goes to `/home/ubuntu/app.log`)

> Provisioners only run when the instance is **created**. After changing `app.py` or the provisioner commands, force a rebuild of just the instance:
>
> ```powershell
> terraform apply -replace="aws_instance.server"
> ```

## Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform/install)
- AWS CLI configured with credentials (`aws configure`)
- An SSH key pair at `~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub`

Check the key exists (PowerShell):

```powershell
Get-Content $HOME\.ssh\id_rsa.pub
```

No key yet? Create one:

```powershell
ssh-keygen -t rsa -b 4096 -f $HOME\.ssh\id_rsa
```

## Usage

```powershell
terraform init
terraform plan
terraform apply
```

Get the instance's public IP:

```powershell
terraform state show aws_instance.server | Select-String 'public_ip\s+='
```

Open `http://<public-ip>` in a browser — you should see `Hello, Terraform!`.

SSH into the instance:

```powershell
ssh -i $HOME\.ssh\id_rsa ubuntu@<public-ip>
```

On the instance, check the app:

```bash
ps aux | grep [a]pp.py      # is it running?
curl localhost               # should print Hello, Terraform!
cat /home/ubuntu/app.log     # Flask output and request log
```

## Clean up

```powershell
terraform destroy
```

## Notes

- The AMI ID is region-specific. If you change `region`, look up the matching Ubuntu AMI:
  ```powershell
  aws ssm get-parameter --region <region> --name /aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id --query Parameter.Value --output text
  ```
- SSH is open to `0.0.0.0/0` for learning purposes — restrict it to your own IP for anything real.
- The app is started with `nohup ... &` rather than a plain `&`. Without `nohup`, the app can be killed when Terraform closes its SSH session after provisioning. It still won't restart after a reboot — a systemd service would be needed for that.
- During `apply` you'll see pip's *"Running pip as the 'root' user"* warning. It's harmless here: Flask installs system-wide on a throwaway instance. For real projects, use a virtualenv or `apt-get install python3-flask` instead.
- State files (`*.tfstate`) are git-ignored since they contain resource IDs and IPs.
