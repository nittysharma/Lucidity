Architecture Diagram (also has PDF): https://drive.google.com/file/d/16rJJccV8pMNGCT9UmnuyLBLiDuIAsSxH/view?usp=sharing

How to understand the solution:

Step 1: We will create an AWS organization, and one of the three accounts can become the parent/master, or we can create a new account. The master account will send invitations to all child accounts, and the child accounts will accept the invitation and become part of the AWS organization.
Step 2: We will create an Ansible node (EC2 machine) in the master account and configure the IAM user in the AWS CLI using AWS configuration. This user will have the role for "ec2:DescribeInstances", "ec2:DescribeRegions", and "organizations:DescribeOrganization".
Step 3: We will use the approach of assuming a role for that. We need to create an IAM role in every child account with a trust relationship with the master account.
Step 4: The Ansible playbook can now be run. This will save the disk utilization report in the format of disk_utilization_<Account-ID>_<Region>.json for every region and every child account.
Step 5: We can move these reports to an Amazon S3 bucket with the date as a prefix for better partitioning.
Step6: From Amazon S3, we can connect to Amazon QuickSight to plot a fancy dashboard with date/region/account filters.

Automation: Amazon EventBridge will schedule running the Ansible playbook every day at midnight (or any other desired time). This will generate data in a new date partition in S3, and QuickSight will have fresh data every morning in addition to historical data.

Answers to specific queries:

How would you centralize access and management of the 3 AWS accounts? Response: Take all child accounts as part of the AWS organization. You can nominate one of these accounts as the master if a master account is not available. Create one EC2 machine in the master account and install Ansible on this machine. This will be our Ansible node. The Ansible playbook is also in this repository. Now, create an IAM user with the following permissions:
{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": [ "ec2:DescribeInstances", "ec2:DescribeRegions" ], "Resource": "" }, { "Effect": "Allow", "Action": "organizations:DescribeOrganization", "Resource": "" } ] }

We will use the Assume Role approach to get access to other AWS accounts that are part of this organization. To do so, we need to create an IAM role in each account with a trust relationship with the master account. Instead of using the root account, you can use the specific user that was created in the master account in the previous step.

{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Principal": { "AWS": "arn:aws:iam::<MasterAccount-ID>:root" }, "Action": "sts:AssumeRole", "Condition": {} } ] }

Now we can run the Ansible playbook using the command "ansible-playbook myansible.yml" from the terminal.

This is how we can consolidate all data in a single account using the AWS organization and the Assume Role approach.

How would you aggregate the collected data from all accounts into a single, easily digestible format? We are creating a .json file with specific fields, which can be controlled based on the format the business wants. The current playbook will create a file in this format: disk_utilization_<Account-ID>_<Region>.json. We are running the task in a loop for every child account and every region. So, after running this playbook, we will have multiple files in the directory "/home/ec2-user/output" (you can change this path).

How would your solution scale if the company acquires more companies and AWS accounts in the future? This is a two-step process. First Step: We can use a landing zone (control tower) to run a CloudFormation template while adding a child account to the organization. When the parent company acquires a new company and adds it to the organization, a CloudFormation template will automatically run and create an IAM Role that will be assumed by the parent. Second Step: Instead of hard-coding the child account IDs, we can run an AWS CLI command to get all AWS account IDs that are children, including those acquired later. This will ensure that every subsequent run of the playbook will also consider new accounts added to the organization using the "list-children" command.

This will not only help when we add a new child account, but it will also work when we remove a child account from the organization, meaning a company is no longer part of the parent company.
