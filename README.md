# AWS CloudFormation Templates

A set of CloudFormation templates for practicing common AWS building blocks: S3, static website hosting, IAM, RDS, a VPC with a bastion host, an ALB with web servers, and an Auto Scaling group.

| File | What it builds |
|------|----------------|
| `s3-bucket.yaml` | A basic S3 bucket |
| `s3-static.yaml` + `index.html` | S3 static website hosting with a public-read bucket policy |
| `iam.yaml` | An IAM user, group, role, and policy |
| `rds.yaml` | A MySQL RDS instance |
| `vpc.yaml` | A VPC with public and private subnets, a bastion host, and two private app servers |
| `ec2.yaml` | A VPC with an Application Load Balancer and two web servers |
| `asg.yaml` | A VPC with an Auto Scaling group and a CPU alarm |

> **Cost warning:** The ALB, EC2 instances, and RDS instance cost money while they run. Deploy one VPC-based stack (`vpc`, `ec2`, `asg`) at a time and delete stacks when you're done.

## Prerequisites

1. Install the AWS CLI and run `aws configure` (or `aws sso login`) with a user or role that can create these resources.
2. Pick one region and stick with it. The AMI ID in the EC2 templates (`ami-0d418c701e6d800c4`) is region-specific, so confirm it exists in your region first.
3. Optionally validate a template before deploying:
   ```
   aws cloudformation validate-template --template-body file://vpc.yaml
   ```
4. The basic deploy command is below. Use a different stack name for each file.
   ```
   aws cloudformation deploy --template-file <file>.yaml --stack-name <name>
   ```

---

## s3-bucket.yaml

1. Bucket names are globally unique. If `bigger-bucket-beyond-too-big` is taken, change it.
2. Deploy:
   ```
   aws cloudformation deploy --template-file s3-bucket.yaml --stack-name s3-bucket
   ```
3. Check that the bucket exists with `aws s3 ls`.

## s3-static.yaml and index.html

**Before deploying**

1. The template sets all four `PublicAccessBlockConfiguration` flags to `false` (`BlockPublicAcls`, `BlockPublicPolicy`, `IgnorePublicAcls`, `RestrictPublicBuckets`) so the public bucket policy can be applied. Bucket names are globally unique, so change `static-website-test-bucket-ea` if it's taken.
2. Check that account-level Block Public Access isn't on (S3 console, Block Public Access settings for this account). If it is, it overrides the bucket settings and the policy will be rejected.

**Deploy**

```
aws cloudformation deploy --template-file s3-static.yaml --stack-name s3-static
```

**After deploying**

1. Upload the page:
   ```
   aws s3 cp index.html s3://static-website-test-bucket-ea/
   ```
2. Open `http://static-website-test-bucket-ea.s3-website-<region>.amazonaws.com`. This endpoint is HTTP only. For HTTPS or a custom domain, add CloudFront (and Route 53 for the domain).

**Before deleting the stack**

Empty the bucket, since CloudFormation can't delete a non-empty bucket:

```
aws s3 rm s3://static-website-test-bucket-ea --recursive
```

## iam.yaml

1. Deploy with the capability flag, since the resources have custom names:
   ```
   aws cloudformation deploy --template-file iam.yaml --stack-name iam --capabilities CAPABILITY_NAMED_IAM
   ```
2. **After deploying:** `EricTemp` has no console password or access keys, so it can't be used yet. If you need it, create a console password or access keys in IAM and turn on MFA.
3. Delete the stack when you're done testing. The user has `AdministratorAccess` and the role has `PowerUserAccess`, so these are for testing only.

## rds.yaml

**Before deploying**

1. Replace the `<YOUR_PASSWORD>` placeholder in `MasterUserPassword` with your own password. Don't commit the real one.
   - Safer options: use a `NoEcho` parameter, or set `ManageMasterUserPassword: true` so RDS stores the password in Secrets Manager.
2. Check that the engine version is still available:
   ```
   aws rds describe-db-engine-versions --engine mysql --query 'DBEngineVersions[].EngineVersion'
   ```
   MySQL 8.0 is aging out of RDS standard support, so `8.0.35` may be rejected. If it is, pick a version from that list.
3. The template has no VPC settings, so the instance lands in your default VPC with the default security group.

**Deploy**

```
aws cloudformation deploy --template-file rds.yaml --stack-name rds
```

RDS takes about 5 to 10 minutes.

**After deploying**

1. Get the endpoint:
   ```
   aws rds describe-db-instances --db-instance-identifier MyNewRDS --query 'DBInstances[0].Endpoint.Address'
   ```
2. Add an inbound rule for port 3306 on the security group, from your IP or from a security group you connect from. By default nothing can reach the database.
3. Connect:
   ```
   mysql -h <endpoint> -u admin -p
   ```
4. Create an application user instead of using `admin` for everything.
5. Backups are off (`BackupRetentionPeriod: 0`). Set it to 1 or more if the data matters.

## vpc.yaml (bastion + private app servers)

**Before deploying**

1. Create the key pair `Bastian_Host_Keys` in this region:
   ```
   aws ec2 create-key-pair --key-name Bastian_Host_Keys --query KeyMaterial --output text > Bastian_Host_Keys.pem
   chmod 400 Bastian_Host_Keys.pem
   ```
2. Replace the `<YOUR_PUBLIC_IP>` placeholder in the bastion security group's `CidrIp` with your current public IP (keep the `/32`). Don't commit your real IP.

**Deploy**

```
aws cloudformation deploy --template-file vpc.yaml --stack-name vpc
```

**After deploying**

1. Load the key into your agent:
   ```
   ssh-add Bastian_Host_Keys.pem
   ```
2. Get the bastion's public IP and the private IPs of App1 and App2 from the EC2 console.
3. Reach App1 through the bastion:
   ```
   ssh -A -J ec2-user@<bastion-public-ip> ec2-user@<app1-private-ip>
   ```
4. From App1, test the ICMP rule:
   ```
   ping <app2-private-ip>
   ```
   It should work, since App2's security group allows ICMP from App1's.

**Notes**

- The private subnets have no NAT gateway, so the app instances can't reach the internet. That's expected here.
- App2 has no key pair, so you can't SSH to it. The ping test is the only access it's built for.

## ec2.yaml (ALB + two web servers)

**Before deploying**

`MyLoadBalancer` has `DependsOn: AttachGateway`, so the load balancer waits for the internet gateway to be attached. Without it, the first deploy can fail.

**Deploy**

```
aws cloudformation deploy --template-file ec2.yaml --stack-name ec2
```

**After deploying**

1. The template has no `Outputs`, so get the ALB address:
   ```
   aws elbv2 describe-load-balancers --query 'LoadBalancers[].DNSName'
   ```
2. Open that DNS name in a browser. Refresh a few times and you should alternate between "Hello from My Web Server 1A" and "2C".
3. In the EC2 console, check Target Groups and confirm both targets show healthy (this can take a minute or two).
4. Consider adding an `Outputs` section with the ALB DNS name so you don't have to look it up.

## asg.yaml (Auto Scaling group + CPU alarm)

**Deploy**

```
aws cloudformation deploy --template-file asg.yaml --stack-name asg
```

There's no load balancer in this one.

**After deploying**

1. Confirm two instances launched (EC2 console, Auto Scaling Groups).
2. Browse to each instance's public IP to see "Hello from Auto Scaling Instance".
3. Check the `HighCPUAlarm` in CloudWatch.

**Testing scale-out**

The template gives you no shell access: no key pair, no SSM role, and port 22 isn't open. To test scale-out, add a key pair and an SSH rule from your IP, or an SSM instance profile. Then run a CPU load tool (for example `stress-ng`) and watch the group grow toward the max of 3.

**Notes**

There's no scale-in policy, so instances won't shrink back down on their own. Add a scale-in policy and alarm if you want that.

---

## Cleanup

Delete stacks when you're done, newest first:

```
aws cloudformation delete-stack --stack-name <name>
```

- Empty the static website bucket first.
- Delete the `Bastian_Host_Keys` key pair if you no longer need it.
