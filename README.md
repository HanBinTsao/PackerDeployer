
# Use packer to deploy image to Amazon AWS

Use a AWS EC2 "t2.micro" instance to build the image.

A builder is a component of Packer that is responsible for creating a machine and turning that machine into an image.
In this case, we're only configuring a single builder of type amazon-ebs.
This builder builds an EBS-backed AMI by launching a source AMI, provisioning on top of that, and re-packaging it into a new AMI.

## Install packer

### Linux

Download binary file to your `/usr/local/bin`

### Windows

use choco command by microsoft windows to install packer

```powershell
PS C:\> choco install packer
Chocolatey v0.10.11
Installing the following packages:
packer
By installing you accept licenses for the packages.
Progress: Downloading packer 1.3.3.20181213... 100%

packer v1.3.3.20181213 [Approved]
packer package files install completed. Performing other installation steps.
The package packer wants to run 'chocolateyInstall.ps1'.
Note: If you don't run this script, the installation will fail.
Note: To confirm automatically next time, use '-y' or consider:
choco feature enable -n allowGlobalConfirmation
Do you want to run the script?([Y]es/[N]o/[P]rint): Y

Removing old packer plugins
Downloading packer 64 bit
  from 'https://releases.hashicorp.com/packer/1.3.3/packer_1.3.3_windows_amd64.zip'
Progress: 100% - Completed download of C:\Users\tbsaa\AppData\Local\Temp\chocolatey\packer\1.3.3.20181213\packer_1.3.3_windows_amd64.zip (26.22 MB).
Download of packer_1.3.3_windows_amd64.zip (26.22 MB) completed.
Hashes match.
Extracting C:\Users\tbsaa\AppData\Local\Temp\chocolatey\packer\1.3.3.20181213\packer_1.3.3_windows_amd64.zip to C:\ProgramData\chocolatey\lib\packer\tools...
C:\ProgramData\chocolatey\lib\packer\tools
 ShimGen has successfully created a shim for packer.exe
 The install of packer was successful.
  Software installed to 'C:\ProgramData\chocolatey\lib\packer\tools'

Chocolatey installed 1/1 packages.
 See the log for details (C:\ProgramData\chocolatey\logs\chocolatey.log).
```

## Create IAM user in AWS (optional)

Create new user to deploy, apply `aws_packer_policy.json` for new user.

## Edit packer json file

use an AWS credentials file to specify your credentials

The default location is `$HOME/.aws/credentials` on Linux and OS X,
or `"%USERPROFILE%.aws\credentials"` for Windows users.

You can optionally specify a different location in the configuration by setting the environment with the `AWS_SHARED_CREDENTIALS_FILE` variable.

and check variable in json config file:
profile
region
owners
instance_type
ami_name

## Validate json file

```powershell
 λ packer
Usage: packer [--version] [--help] <command> [<args>]

Available commands are:
    build       build image(s) from template
    fix         fixes templates from old versions of packer
    inspect     see components of a template
    validate    check that a template is valid
    version     Prints the Packer version


λ packer validate aws-packer-example.json
Template validated successfully.
```

## Build the image from this template (use variable)

```powershell
packer build -var 'aws_access_key=YOUR ACCESS KEY' -var 'aws_secret_key=YOUR SECRET KEY' aws-packer-example.json
```

## Build the image from this template (for the credentials file)

create file name call credentials.json

```json
{
    "aws_access_key": "YOUR AWS ACCESS KEY HERE",
    "aws_secret_key": "YOUR AWS SECRET KEY HERE"
}
```

build packer image with variable file in shell,
use `-var-file` option to hide secret file.

```powershell
packer build -var-file credentials.json aws-packer-example.json
```

output:

```powershell
amazon-ebs output will be in this color.

==> amazon-ebs: Prevalidating AMI Name: aws-packer-example 1544819098
    amazon-ebs: Found Image ID: ami-032f516e93380b8e6
==> amazon-ebs: Creating temporary keypair: packer_xxxxxxxxx_xxxxxx
==> amazon-ebs: Creating temporary security group for this instance: packer_xxxxxxxxx_xxxxxx
==> amazon-ebs: Authorizing access to port 22 from 0.0.0.0/0 in the temporary security group...
==> amazon-ebs: Launching a source AWS instance...
==> amazon-ebs: Adding tags to source instance
    amazon-ebs: Adding tag: "Name": "Packer Builder"
    amazon-ebs: Instance ID: i-043a8ed915d288a8d
==> amazon-ebs: Waiting for instance (i-043a8ed915d288a8d) to become ready...
==> amazon-ebs: Using ssh communicator to connect: xxx.xxx.xxx.xxx
==> amazon-ebs: Waiting for SSH to become available...
==> amazon-ebs: Connected to SSH!
==> amazon-ebs: Stopping the source instance...
    amazon-ebs: Stopping instance, attempt 1
==> amazon-ebs: Waiting for the instance to stop...
==> amazon-ebs: Creating unencrypted AMI aws-packer-example 1544819098 from instance i-043a8ed915d288a8d
    amazon-ebs: AMI: ami-0bf2f2638f1dbcf5d
==> amazon-ebs: Waiting for AMI to become ready...
==> amazon-ebs: Terminating the source AWS instance...
==> amazon-ebs: Cleaning up any extra volumes...
==> amazon-ebs: No volumes to clean up, skipping
==> amazon-ebs: Deleting temporary security group...
==> amazon-ebs: Deleting temporary keypair...
Build 'amazon-ebs' finished.

==> Builds finished. The artifacts of successful builds are:
--> amazon-ebs: AMIs were created:
ap-northeast-1: ami-0bf2f2638f1dbcf5d
```

### Verify packer build images

```shell
aws ec2 describe-images --image-ids ami-03d2fc6373990e64e
```



## AWS block device mappings

use file `aws-packer-block_device_mappings-example.json` to create tree 10 GB ebs volume.

you can add tag with volume:

```json
"tags": {
            "OS_Version": "Ubuntu",
            "Release": "Latest",
            "Base_AMI_Name": "{{ .SourceAMIName }}",
            "Extra": "{{ .SourceAMITags.TagName }}"
        },
```



## Packer with Ansible

`docker-packer-builder.json` file

## Drone CI

add `drone.yml` file

### add drone secret

```shell
drone secret add \
  -repository hbtdocker/aws-packer-example \
  -image appleboy/drone-packer \
  -event push \
  -name aws_access_key_id \
  -value `YOUR SECRET HERE`
```

add drone secret with file

```shell
drone secret add \
  --name ssh-key \
  --value @./credentials.json \
  --image appleboy/drone-ssh \
  --repository packerdeployer
```

## How to find OwnerId in aws cli

for example, find the windows ownerid

```bash
aws ec2 describe-images --owners amazon --filters "Name=platform,Values=windows" "Name=root-device-type,Values=ebs" "Name=architecture,Values=x86_64" "Name=name,Values=*Windows_Server-2008"Name=name,Values=*Windows_Server-2008-SP2*English-64Bit-Base*" | jq -r '.Images[] | "\(.OwnerId)\t\(.Name)"'
```

for more info, refer [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/finding-an-ami.html)

## add post process

syntax reference

```json
    "variables": {...},
    "builders": [{
    "provisioners": [{
    "type": "shell",
    "inline": [
      "sleep 30",
      "sudo apt-get update",
      "sudo apt-get install -y redis-server"
      ]}],
     "post-processors": ["vagrant"]
    }
```

## clean up test instance

```shell
aws ec2 terminate-instances --instance-ids "YOUR INSTANCES ID"
aws ec2 deregister-image --image-id ami-0123456789
aws ec2 delete-snapshot --snapshot-id snap-9876543210
```