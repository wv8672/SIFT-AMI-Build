# SIFT_AMI_Builder
##### Contains JSON files to build a SIFT Forensics Workstation AWS AMI from an exported Ubuntu Desktop .ova

-------------------------------------------------------------------------------------------------------

![alt text](https://camo.githubusercontent.com/88f7a671578a17f5e14d2227fb2d7fca9fa0b1f7/68747470733a2f2f6469676974616c2d666f72656e736963732e73616e732e6f72672f696d616765732f736966742e706e67)

--------------------------------------------------------------------------------------------------------

## Part 1: Create local SIFT VM and export as .ova


#### Step 1: Download the Ubuntu Desktop ISO image

From Browser:
Visit & download: http://releases.ubuntu.com/16.04/ubuntu-16.04.6-desktop-amd64.iso

#### Step 2: Configure the VMware Workstation VM 

From VMware Workstation:
* Select create new virtual machine 
* Select typical / recommended  & next 
* Select i will install the operating system later & next 
* Select linux as the guest operating system & ubuntu 64 bit as version 
* Select name/location/disk size. (unrelated as this will be a AMI ) 
* Note: my settings: name:ubuntu 64-bit, location: vmware/ubuntu 64-bit, disk size:20gb & next
* Select store virtual disk as single file
* Select finish
* From home menu, highlight the newly created vm ex: ubuntu 64-bit
* Once highlighted, visit the VM menu bar & choose settings from the dropdown (vm → settings)
* Select CD/DVD and select use ISO image file 
* Navigate to the ubuntu-16.04.6-desktop-amd64.iso file that was downloaded & open
* Select save & start the operating system 

#### Step 3: Install the SIFT binaries

From Terminal:
* as root user
```
apt update && apt upgrade
wget https://github.com/teamdfir/sift-cli/releases/download/v1.8.5/sift-cli-linux
wget https://github.com/teamdfir/sift-cli/releases/download/v1.8.5/sift-cli-linux.sha256.asc
```

#### Step 4: Validate the signature file

From Terminal:
* as root user
```
gpg --keyserver hkp://pool.sks-keyservers.net:80 --recv-keys 22598A94
gpg --verify sift-cli-linux.sha256.asc
shasum -a 256 -c sift-cli-linux.sha256.asc
```

#### Step 5: Move binaries and change permissions
From Terminal:
* as root user
```
mv sift-cli-linux /usr/local/bin/sift
chmod 755 /usr/local/bin/sift
cd /usr/local/bin 
```

#### Step 6: Run SIFT Install 

From Terminal:
* as root user
```
sift install
```

#### Step 7: Install openssh-server and allow SSH access 

From Terminal:
* as root user 
```
apt update 
apt install openssh-server
ufw allow 22
service ufw start
```
* [exit terminal and power off machine]

#### Step 8: Export VM as .ova 

From VMware Workstation:
* File → export to OVF
* Save file as .ova  |  ex: Ubuntu 64-bit.ova
 
--------------------------------------------------------------------------------------

## Part 2:  Build the AWS AMI from the .ova file 

#### Step 1: Configure AWS CLI

From AWS Console: 
* Navigate to IAM resource
* Select users from left hand side bar 
* Select add user 
* Supply username & check programmatic access type 
* Add user to new or existing group with AdministratorAccess policy attached
* Select Create User 
* Important: Download .csv file [this contains AWS creds]

From Terminal:
```
aws configure
(Add AWS Access Key ID from the downloaded IAM user .csv file)
(Add AWS Secret Access Key from the downloaded IAM user .csv file) 
(Add a default region name. Ex: us-east-2(ohio) )
```

#### Step 2: Upload the .ova file to an AWS S3 Bucket 

From AWS Console:
* Select the S3 resource 
* Select Create Bucket and supply name & region 
* Ex: us-east-2(ohio)

#### Step 3: Create trust-policy.json

From Terminal:
* run as root
```
mkdir sift-ami
cd sift-ami
touch trust-policy.json
vim trust-policy.json
```
* NOTE: This IAM role allows access to the file in the S3 bucket created in step 2

#### Step 4: RUn AWS CLI command to build vmimport IAM role 

From Terminal / AWS CLI:
```
cd sift-ami
aws iam create-role --role-name vmimport --assume-role-policy-document file://sift-ami/trust-policy.json
```

#### Step 5: create role-policy.json

From Terminal:
```
cd sift-ami
vim role-policy.json file
```

#### Step 6: Apply the vmimport role

From Terminal / AWS CLI:
```
cd sift-ami
aws iam put-role-policy --role-name vmimport --policy-name vmimport --policy-document file://sift-ami/role-policy.json
```

#### Step 7: Push the .ova file to AWS S3 

From Terminal / AWS CLI:
```cd sift-ami
aws s3 cp capstone-sift-workstation-ami.ova s3://capstone-sift-workstation-ami/
```

#### Step 8: Create containers.json 

From Terminal:
```cd sift-ami
vim containers.json
```

#### Step 9:  Build SIFT AMI

From Terminal / AWS CLI:
```
aws ec2 import-image --description "SIFT AMI" --disk-containers file://sift-ami/containers.json
```

#### NOTE: At this point the AMI is created:
The EC2 resource AMI section within the selected region contains the new SIFT AMI. This is what will be used to automatically build SIFT EC2’s in the incident response  Python scripts. 

### AMI ID: ami-0faead58612458329

