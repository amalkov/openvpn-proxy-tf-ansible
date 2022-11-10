# OpenVPN with Terraform and Ansible on AWS

This repository is an example of an _Infrastructure as Code_ devOps project. It provides fully automatic deployment of a private VPN on your AWS account using Terraform and Ansible.

## Prerequisites

### 1) Install AWS CLI

- On MacOS: `brew install awscli`

For other Operating Systems see https://docs.aws.amazon.com/cli/latest/userguide/installing.html

### 2) Configure an admin user

1.  Go to https://console.aws.amazon.com/iam/home#/home
2.  Choose a username _(e.g. terraform-vpn)_ and give programmatic access.
3.  Add exiting policy: _AdministratorAccess_
4.  Download the credentials and configure a profile in aws-cli

```bash
aws configure --profile terraform-vpn
aws iam get-user --profile terraform-vpn
```

### 3) Create a ssh key-pair to access the OpenVPN instance

```bash
ssh-keygen -t rsa -C "your.email@example.com" -b 4096 `
chmod 600 ~/.ssh/vpn
```

### Open Port 8888 for Proxy Server

In **EC2 Management Console** go to **Security Groups** and create a group that allows connections on ports 22 and 8888. Apply this group to proxy instance. Also, it is highly recommended to limit IPs of users by configuring Source settings. Proxies that are left wide open will be discovered by proxy scanners and used by random people. This can result in Amazon charges for network bandwidth.

## Configuration

### 1) Modify the config file as you wish _/config.json_

```json
{
  "REGION": "us-west-1",
  "PROFILE": "terraform-vpn",

  "VPN_INSTANCE_TYPE": "t2.micro",
  "VPN_AMI": "ami-02ea247e531eb3ce6",
  "VPN_SSH_PUBLIC_KEY": "~/.ssh/vpn.pub",
  "VPN_SSH_PRIVATE_KEY": "~/.ssh/vpn",
  "OVPN_PORT": "1194",

  "VPC_CIDR": "172.20.0.0/16",
  "VPC_CIDRS": {
    "public": "172.20.3.0/24",
    "private": "172.20.1.0/24"
  }
}
```

### 2) Modify the default vars of the openvpn ansible role as you wish _/ansible/roles/openvpn/default/main.yml_

```yml
ovpn_cidr: 10.3.0.0/24
ovpn_network: 10.3.0.0 255.255.255.0
ovpn_push_routes:
  - 172.20.0.0 255.255.0.0

ca_dir: /home/ubuntu/ca

ca_key_country: BE
ca_key_province: BR
ca_key_city: Brussels
ca_key_org: MyOrganization
ca_key_email: your.email@organization.org
ca_key_org_unit: MyOrganizationalUnit
ca_key_name: vpn_server
```

## Setup

### 1) Add the AWS credentials to your environment

```bash
export AWS_ACCESS_KEY_ID="YOUR_AWS_KEY_ID"
export AWS_SECRET_ACCESS_KEY="YOUR_AWS_SECRET"
export AWS_DEFAULT_REGION="YOUR_AWS_REGION"
```

### 2) Bootstrap the infrastructure

```bash
cd terraform
terraform init
terraform plan --var-file ../config.json
terraform apply --var-file ../config.json
```

### 3) Wait till the EC2 is ready

### 4) Install OpenVPN on the EC2 Instance

This will download a zip file with client openvpn configuration and keys to your host.

```bash
cd ansible

# This will also add a client
ansible-playbook -i inventory openvpn_install.yml -e "username=alex" -e "output=/tmp/alex_vpn.zip"
```

#### Error grub-efi-amd64-bin
`grub-efi-amd64-signed : Depends: grub-efi-amd64-bin (= 2.06-2ubuntu7) but 2.06-2ubuntu10 is to be installed`
ssh to the server and run
`apt-get upgrade --with-new-pkgs`
to fix issue
`sudo apt --only-upgrade install grub-efi-amd64-signed`

#### Error Unable to restart dnsmasq.service
to check the state
`systemctl status dnsmasq.service`
Error
`failed to create listening socket for port 53: Address already in use`

```
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo systemctl mask systemd-resolved
To undo what you did:

sudo systemctl unmask systemd-resolved
sudo systemctl enable systemd-resolved
sudo systemctl start systemd-resolved
```


### 4a) Add an additional client to the VPN

This will download a zip file with client openvpn configuration and keys to your host.

```bash
cd ansible
ansible-playbook -i inventory proxy_install.yml

```

### 5) Add an additional client to the VPN

This will download a zip file with client openvpn configuration and keys to your host.

```bash
cd ansible
ansible-playbook -i inventory openvpn_add_client.yml -e "username=alex" -e "output=/tmp/alex_vpn.zip"

```


## Reprovision the EC2

If you want to recreate the vpn server with a new IP adress and new correct configuration, run these commands:

```bash
# taint the ec2 instance and ansible inventory generation script, this means it will be destroyed and recreated

cd terraform
terraform taint aws_instance.vpn
terraform apply --var-file ../config.json -auto-approve

# wait till the instance get up ...

# provision again with ansible
cd ../ansible
ansible-playbook -i inventory openvpn_install.yml -e "username=alex" -e "output=/Users/brmm/Desktop/alex_vpn.zip"
```

## DNS Problems:

- If you use tunnelblick on Mac on Sierra or higher you might have DNS issues with `allowChangesToManuallySetNetworkSettings` see this [github issue](https://github.com/Tunnelblick/Tunnelblick/issues/401)
