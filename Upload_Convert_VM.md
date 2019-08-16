# Uploading a Virtual Machine and Converting it to AMI Format on AWS

There are a variety of posts online regarding how to convert a Virtual Machine in VMDK format to AMI format on Amazon Web Services.  Many of these posts are outdated.

As of 2019, AWS has [tools](https://docs.aws.amazon.com/vm-import/latest/userguide/vmimport-image-import.html#import-vm) for doing this fairly easily.  However, I have found that their documentation is lacking in certain regards, so what I've learned is listed below.

First, the AWS *vmimport* tools supposedly will work directly on a .VMDK formatted VM.  However, I have found that it's not true if the .VMDK image was created with VirtualBox.  Some users say that the .VMDK image must be from VMWare.

Second, I have found that a .VMDK VirtualBox image that is converted to an .OVA (using OVF version 2.0 format) can be successfully converted to .AMI image on AWS.

Third, the readily available, current versions of Ubuntu are using kernel-5 as of August 2019, but that kernel is not supported by AWS EC2.  Rather, the kernel must be version 4 or earlier.

Fourth, whatever local Virtual Machine you intend to import to AWS must have SSHD configured and running so that you eventually will be able to connect to that machine.

Below I have the detailed steps I went through for converting a local VM in .VMDK format on VirtualBox to a functioning .AMI image on AWS.  This is written for someone who is comfortable on Linux, but is a non-expert. So there's lots of detail.

### A. Getting Your Local VM Ready
1. I'm starting from the point when you have your Linux guest machine successfully installed on VirtualBox.  It should be in VMDK format, and not VDI.  One can convert from VDI to VMDK using VirtualBox, but I have not tested whether that works for the eventual conversion to AMI on AWS.  You'll also need to have your Linux guest machine configured for **sshd** to be running when it is booted. Also, you should not have Guest Additions installed on your Linux guest machine.  
2. Shut down the Linux guest machine.
3. On the VirtualBox Manager Console, select on the Linux guest machine that you'll convert. Select Machine -> Export to OCI... -> Format -> Open Virtualization Format 2.0.  The File shows that it will be exported in OVA format, which is a compressed form of OVF2.0.  I set the MAC Address Policy to "include only NAT network adapter MAC addresses".  I also checked "Write Manifest file", but I don't know whether that's important.
4. Converting to OVA was important, as earlier attempts at directly converting the VMDK image to AMI on AWS failed.  Others have said that direct VMDK->AMI conversion requires a VMWare VMDK image and does not work with VirtualBox VMDK images.
5. Once the OVA image has been written to a local drive, you'll want to split it into smaller pieces so that any uploading errors can be corrected on just the piece that failed.  I decided to split my OVA image into 200 MB chunks.  On a MacOS or Linux host machine, you'd use `$ split -b 200mb filename.ova` on the Terminal.  That will create a series of files: *xaa, xab, ... , xaf*.  The number of files depends on how large your OVA file was and how big your pieces are.  You should be aware that by default, a file (e.g. xab) must finish uploading in less than an hour from when it starts.  See Section C7 below.  You should keep that in mind, as well as your internet upload speed, when you choose how big your image chunks will be.

### B. Setting up AWS and AWS Command Line Interface

1. You must set up an [AWS Account](https://aws.amazon.com).  During this process, you will:  
  a. set up a username and password, which are used for logging onto the AWS website.  
  b. You'll want to create a second AWS user which does not have root priviledges, [using AWS Identity and Access Management (IAM)](https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started_create-admin-group.html).  This user will have its own username and password, as well as
  an *AWS Access Key ID* and an *AWS Secret Access Key*.  These are used for accessing AWS via the Command Line Interface.   
  c. Store all this information somewhere safe, such as in a Password Manager.
2. Install AWS Command Line Interface Tools (CLI) on your local machine.  I did this via *pip*.  From Terminal: `$ pip install awscli --upgrade --user`.  The installation was performed on my MacOS at *~/.local/bin/*, so I added this to my PATH in my .bash_profile file.
3. On Terminal, type `$ pip3 list -o` to determine if *awscli* is up to date.
3. The next consideration is which AWS Region/Site you will use.  This is important because each Site operates more-or-less independently from the others.  While data can be transferred between Sites, you'll want to upload your data and launch your machine all from a single Site. The one with the quickest access for me (which affects upload time) is *us-west-1*.
4. Now connect to AWS via CLI.  On Terminal: `$ aws configure` You'll enter your *AWS Access Key ID*, your *AWS Secret Access Key*, your Default region name (input what you chose in step 3--e.g. 'us-west-1'), and the Default output format, which is typically either 'json' or the more abbreviated 'text'.

### C. Creating an S3 Bucket and Uploading your VM
1. When we upload our VM to AWS, we need to put it somewhere.  That is done at Amazon Simple Storage Service (S3). Using S3 requires that you separately [sign up](https://docs.aws.amazon.com/AmazonS3/latest/gsg/SigningUpforS3.html) for it.  Make sure you do this in the AWS region (e.g. aws-west-1) where you'll be doing your work.
2. Once you receive an email from AWS saying that your S3 account is ready, you can create an S3 bucket for receiving uploaded files.  I did this via the AWS S3 Console webpage.  Click on Create Bucket.  The bucket must be a unique name across all AWS S3, so a simple name like 'bucket' won't suffice.  Be sure to select the Region/Site where you'll do your work/uploading.  I chose to Block All Public Access -> Next -> Create Bucket.
3. Go back to the Terminal, and check that you can see this newly created bucket: `$ aws s3 ls`.  You should see it whether or not you're logged into the same AWS Region as where the S3 bucket resides.
4. The simplest way to upload a file to your S3 Bucket is to use the Terminal command: `$ aws s3 cp file_name s3://bucket_name`.  This will automatically upload the file using a "storage-class" of STANDARD.  There are other storage classes, but I won't cover that here.
5. Because we split our VM.ova image into multiple files, we'll need to upload those parts in a way that *ETag*-s are generated, as they will be needed to reconstruct the intact file on AWS's side once the uploading is finished.  AWS will need to keep track of the various files we'll upload, so we create an *upload-id* first using the Terminal command: `$ aws s3api create-multipart-upload --bucket bucket_name --key UbuntuServer.ova`.  The `--key` parameter is what you'll call your machine.
6. The Terminal will return an output such as this:  
```bash
    "ServerSideEncryption": "AES256",  
    "Bucket": bucket_name,  
    "Key": "UbuntuServer.ova",  
    "UploadId":   "kkvJ.Tu7bCoXVUXsb0bYnIzEg46nAYkscLwsgEe0NVn_ZW63JNh5QmZ3FQITwZ9k7Fk4GsSbIipvyZUHcoi2mI4sup_8R0wNHetwTqzGAfpDZ_._b7AtYyEBRAqdgB.0"  
```
  Be sure to save this information in a file.
7. If your OVA image is only broken up into a few parts (say 3 or 4), you could serially upload them.  Or you could upload them in parallel if they are collectively small enough and your upload bandwidth is sufficiently high.  See section A5.  To upload a single image chunk, the command is `$ aws s3api upload-part --key UbuntuServer.ova --part-number 1 --body xaa --bucket bucket_name --upload-id UPLOADID`.  The BODY is simply the name of the sub-file being uploaded (e.g. xaa, xag, etc.).  THE PART is the index of the BODY file.  So xaa would have PART=1, xag would have PART=7, and xaz would have PART=26.  As the command finishes, it will return an *ETag* to you, which will be important later.  You can also obtain all the *ETag*s later after the uploading is finished.  By the way, there's a *--content-md5* option for confirming that the upload was received without any missing bits.  I did not use that, but [there's AWS documentation on *upload-part*](https://docs.aws.amazon.com/cli/latest/reference/s3api/upload-part.html).
8. Rather than micromanaging the upload, it's better to use a shell script.  I include mine here, which is called *awsSerialUpload.sh*.
```bash
#!/bin/bash
BUCKET=bucket_name
KEY=UbuntuServer.ova
UPLOADID=7kOwniA8ar2XGRBHoY744FVZA.0IlIoJZCI5YTSmeqvDzUrhpuiIkRlUlvpAgXBuUTeDV7Ik9NLPB5rA02pAy.Jw4CrMqrRjLkI7DWwSKZwfMakFVBAl_ZE3VcXIPlrP
PART=1
files="xaa xab xac xad xae xaf xag xah"
for BODY in $files
do
    aws s3api upload-part --key $KEY --part-number $PART --body $BODY --bucket $BUCKET --upload-id $UPLOADID >> ETags_Inprogress.txt
    ((PART++))
done
```
You'll want to insert your values of *BUCKET, KEY, UPLOADID,* and *files*. Place *awsSerialUpload.sh* in the same folder as your image chunks 'xaa', 'xab', etc., change the permissions `$ chmod 755 awsSerialUpload.sh`, and then execute the script `$ ./awsSerialUpload.sh`.
9. After the script finishes, we'll want to make sure that AWS S3 received all the files.  If you go to your bucket on the AWS S3 Console webpage, you won't see anything at this point.  So we'll use the *s3api list-parts* command: `$ aws s3api list-parts --bucket bucket_name --key KEY --upload-id UploadId`.  The returned output should contain each part that you uploaded.  If not, then re-upload any missing parts.  It will also contain an *ETag* for each part.
10. In order to re-constitute the original OVA file, we'll need to create a file called *fileparts.json* which contains all the part numbers and their corresponding *ETag*s:
```bash
{
    "Parts": [
        {
            "PartNumber": 1,
            "ETag": "459d4cea937d378fb8c9c5ebc4ce72d7"
        },
        {
            "PartNumber": 2,
            "ETag": "d7a0c3a40c3c7c1c93b94103702dfbc3"
        },
        {
            "PartNumber": 3,
            "ETag": "eb4091624347609409c8febbaa5a1a56"
        }
]}
```
Your fileparts.json file will probably have more parts.
11. We now use the *s3api complete-multipart-upload* command to both reconstitute the original UbuntuServer.ova file from its parts and also to free up S3 resources.  `$ aws s3api complete-multipart-upload --multipart-upload file://fileparts.json --bucket bucket_name --key UbuntuServer.ova --upload-id UPLOADID`.  We will get an output similar to this:
```bash
{
    "ServerSideEncryption": "AES256",
    "Location": "https://bucket_name.s3.amazonaws.com/UbuntuServer.ova",
    "Bucket": bucket_name,
    "Key": "UbuntuServer.ova",
    "ETag": "\"347bee71a760c417a2b96ff0e800671c-7\""
}
```
12. At this point, you should see your equivalent of *UbuntuServer.ova* on the AWS S3 Console/bucket_name webpage.

### D. Converting Your Uploaded OVA Image to an AMI Image
1. We'd like to convert our uploaded Virtual Machine OVA Image into an Amazon Machine Image (AMI).  To do that, we'll need to grant *vmimport* permissions to act on our OVA image.  This involves creating a *trust policy*, and *putting the policy* as described [here](https://docs.aws.amazon.com/vm-import/latest/userguide/vmimport-image-import.html)
2. Let's walk through the process. First, we create a file, *trust-policy.json* which is:
```bash
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Principal": { "Service": "vmie.amazonaws.com" },
         "Action": "sts:AssumeRole",
         "Condition": {
            "StringEquals":{
               "sts:Externalid": "vmimport"
            }
         }
      }
   ]
}
```
You can use the file as-is.  And there's no need to change the Version value or anything.  This indicates that we're allowing the vmie.amazonaws.com to assume the role of *vmimport*.  To create that role, we use the *iam create-role* command: `$ aws iam create-role --role-name vmimport --assume-role-policy-document file://trust-policy.json`.  Successful execution will return a text or json output, which will include the statement `"Effect": "Allow"`.
3. Next, we need to affix that role to our S3 bucket.  We define the rules of *putting* that role in a file called *role-policy.json*:
```bash
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Action":[
            "s3:GetBucketLocation",
            "s3:GetObject",
            "s3:ListBucket"
         ],
         "Resource":[
            "arn:aws:s3:::bucket_name",
            "arn:aws:s3:::bucket_name/*"
         ]
      },
      {
         "Effect":"Allow",
         "Action":[
            "ec2:ModifySnapshotAttribute",
            "ec2:CopySnapshot",
            "ec2:RegisterImage",
            "ec2:Describe*"
         ],
         "Resource":"*"
      }
   ]
}
```
The only thing in this file you'll need to modify is the 2 lines containing  *arn:aws:s3:::bucket_name* where you replace *bucket_name* with your bucket name.  We use the *iam put-role-policy* command to affix this role. `$ aws iam put-role-policy --role-name vmimport --policy-name vmimport --policy-document file://role-policy.json`.
4. The immediate preceding steps only need to be done once.  At this point, we're ready to import our OVA image into an AMI image.  Again, we'll create a file called *containers.json* which will define the job:
```bash
[
  {
    "Description": "UbuntuServer import",
    "Format": "ova",
    "UserBucket": {
        "S3Bucket": "mjkbkt",
        "S3Key": "UbuntuServer.ova"
    }
}]
```
5. `$ aws ec2 import-image --description "UbuntuServer"--disk-containers file://containers.json` which produced the output:
```bash
{
    "Description": "UbuntuServer",
    "ImportTaskId": "import-ami-0b21c8101e749b57d",
    "Progress": "2",
    "SnapshotDetails": [
        {
            "DiskImageSize": 0.0,
            "Format": "OVA",
            "UserBucket": {
                "S3Bucket": bucket_name,
                "S3Key": "UbuntuServer.ova"
            }
        }
    ],
    "Status": "active",
    "StatusMessage": "pending"
}
```
The process of importing will usually take more than 15 minutes, and can take a lot longer, depending on your file size.  In order to monitor the importing process (and to know it's finished), you'll need to "ImportTaskID", which was *import-ami-0b21c8101e749b57d* in my case.
6. `$ aws ec2 describe-import-image-tasks --import-task-ids import-ami-0b21c8101e749b57d` which returned
```bash
{
    "ImportImageTasks": [
        {
            "Description": "UbuntuServer",
            "ImportTaskId": "import-ami-0b21c8101e749b57d",
            "Progress": "6",
            "SnapshotDetails": [],
            "Status": "active",
            "StatusMessage": "validated"
        }
    ]
}
```
The *StatusMessage* returned as *validated*.  What is important is that it was not returned as *deleted* or *internalerror*.  As you monitor the status, it will go through many states which are described [here](https://blog.zhaw.ch/icclab/walk-through-importing-virtual-machine-images-into-ec2/).  Eventually, the returned output of *describe-import-image-tasks* will be:
```bash
{
    "ImportImageTasks": [
        {
            "Architecture": "x86_64",
            "Description": "UbuntuServer",
            "ImageId": "ami-0ef50024524eef9d6",
            "ImportTaskId": "import-ami-0b21c8101e749b57d",
            "LicenseType": "BYOL",
            "Platform": "Linux",
            "SnapshotDetails": [
                {
                    "DeviceName": "/dev/sda1",
                    "DiskImageSize": 789631488.0,
                    "Format": "VMDK",
                    "SnapshotId": "snap-0c05309efb3b713f6",
                    "Status": "completed",
                    "UserBucket": {
                        "S3Bucket": "mjkbkt",
                        "S3Key": "Ubuntu16046Server.ova"
                    }
                }
            ],
            "Status": "completed"
        }
    ]
}
```
Here, the *Status* is *completed*.  In that case, there are also new fields, most notably *ImageID*, which was *ami-0ef50024524eef9d6* and *SnapshotId*, which was *snap-0c05309efb3b713f6*.

### E. Launching AMI Images
1. Now that the image has been successfully converted into an AMI, we can launch it.  If you go to AWS -> EC2 -> Elastic Block Store -> Snapshots, you'll see the image that just completed by its *Snapshot ID*.  If you don't see it, check that your page refers to the correct AWS Region (upper right-hand corner of webpage).  Also, you won't see the image under Images -> AMIs as you might expect.  You also won't see anything under Elastic Block Store -> Volumes at this point.
2. This step you may not need to do.  In my case, I originally set up my S3 bucket in a region where I don't intend to launch my AMI.  So I need to copy it to my desired region.  Although we only see our image by its *Snapshot ID* on the AWS webpage, I've only been able to copy the image to a different region using its *Image ID*.
`$ aws ec2 copy-image --source-image-id "ami-0ef50024524eef9d6" --source-region us-east-1 --region us-west-1 --name UbuntuServer --description UbuntuServer` which returned
` {
    "ImageId": "ami-XXXXXXXXXXXX"
} `
You can see that my S3 bucket was located at the North Virginia Region (us-east-1) and that I moved it to the Northern California Region (us-west-1).  Now, going to the AWS EC2 Console in N. California -> Elastic Block Store -> Snapshots, I see the copied snapshot with a *Snapshot ID* that is new.  I can see in the *Description* the relevant *ImageID*s: ami-0ef50024524eef9d6 and ami-XXXXXXXXXXXX.
3. To convert the snapshot to an image, select on the snapshot -> Actions -> Create Image.  I named mine UbuntuServer, and left everything else as defaults.
4. Under AWS EC2 -> Images -> AMIs. I can now see my AMI.  Just click *Launch* to fire off the instance.  When you do, you'll want to either use a pre-existing SSH key-pair or create one on the fly.  Be sure to store you private key (.pem) in a reliable location on your local machine, such as the *~/.ssh/* folder.  Also make sure that the Security Group being used has *port 22 tcp* open.  In the lower right-hand corner of that page, you'll see the *Public DNS (IPv4)* field populated once the instance is fully launched.  For more information on launching the instance, look [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/launching-instance.html).

### F. Connecting to AMI via SSH
Back to the Terminal, you can ssh to your machine. `$ ssh -v -i path-to-private-key.pem user_name@ec2-54-67-60-89.us-west-1.compute.amazonaws.com`.  Note the last argument to the ssh command.  It starts with 'user_name'.  On an Ubuntu AMI that Amazon has created, user_name is 'ubuntu'.  On other linux flavors, user_name is usually 'ec2-user'.  In my case, user_name is the user account name that I created on my virtual machine while it was still in VirtualBox.  As I'd also created a user password on VirtualBox, I was prompted to enter it at the end of the ssh verification.  Also, the *ec2-54-67-60-89.us-west-1.compute.amazonaws.com* portion of the final ssh argument was the *Public DNS (IPv4)* copied from the AWS EC2 Instances webpage.




### Links
[General Page for Image Importing](https://aws.amazon.com/premiumsupport/knowledge-center/import-server-ec2-instance/)  
[Page Having Lots of Great, Detailed Info, possibly dated](
http://idevelopment.info/data/AWS/AWS_Tips/AWS_Management/AWS_10.shtml)




I'm finding that I can see something in EBS -> Snapshots and EBS -> Volumes on E2 Console.  I copied into us-west-1, and I can only see it there.  It's 10GBi and progress is 0%.
The original one from yesterday I can see on Virginia us-east-1 EC2 Console under "Snapshots" and its progress is 100% and it's "completed". -> Right-click and "Create Image from EBS Snapshot" -> Create.
I now see the AMI image in EC2 Console -> Images -> AMIs.  I "Launch" it.
Each Security Group is datacenter-specific.  So if you copy your image to a new location (e.g. us-west-2), then you must add a Security Group if you want to connect via SSH.
Security group name: launch-wizard-1
Type: SSH, Protocol: TCP, Port Range:22, Source: Anywhere 0.0.0.0/0 (as my IP changes a lot), Description: "SSH for Admin Desktop".

The reason that I could never boot Lubuntu (port 22 connection denied).  There was no sshd_config file in /etc/ssh/ on Lubuntu.  Or was it because my user wasn't "mk"?






**Links**
[Understanding *describe-import-image-tasks* error codes](https://blog.zhaw.ch/icclab/walk-through-importing-virtual-machine-images-into-ec2/)
