Architecture Diagram (also has pdf) :https://drive.google.com/file/d/16rJJccV8pMNGCT9UmnuyLBLiDuIAsSxH/view?usp=sharing
How to understand the solution:
1. Step1 we will create AWS organisation one of three account can become parent/master or we can create new account. Master account will send invitation to all child account and child account will accept invitation and become part of AWS organisation.
2. Step2. We will create on Ansible node(EC2 machine) in master account and configure IAM user in aws cli using aws configuration, this user will have role for   "ec2:DescribeInstances", "ec2:DescribeRegions" "organizations:DescribeOrganization".
3. Step3: we will use approach of assuming role for that we need to create IAM role in every child account. with Trust relastionshop to master account.
4. Step4. Ansible playbook now can be run, this will save disk-utlisation report in format of disk_utilisation_<Account-ID>_<Region>>.json for every region and every child account.
5. We can move these report to Amazon S3 buckect with date as prefix for better partition.
6. From Amazon S3 we can connect Amazon Quicksight to plot Fansy dashboard with date/region/account filter.

Automation:
Amazon eventbridge will schedule runing of Ansible playbook every day at 12midnight(any time). This will generate data in s3 new date partition and Quicksight will have fresh data every morning in addition to historical data.

Answer to very specify queries:
1. How would you centralize access and management of the 3 AWS accounts?
Response: Take all child account as part of AWS organisation, you can nominate one of these account as master if master account not avilable. Create one EC2 machine in master account install ansible on this machine, this will be our Ansible node. Ansible playbook is also in this repository. Now create a IAM user which has permission 
as {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeRegions"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "organizations:DescribeOrganization",
            "Resource": "*"
        }
    ]
}
We will use Assume Role approach to get access of other AWS account part of this Organisation, to do so we need to create IAM role in each account with Trust releationship on master account as instead of root you can use specific user which was created in master account in previous step.
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<MasterAccount-ID>:root"
            },
            "Action": "sts:AssumeRole",
            "Condition": {}
        }
    ]
}

Now we can run ansible playbook using command "ansible-playbook myansible.yml" from terminal, 

This is how we can consolidate all data in single account using AWS organisation and Assume Role approach.

2. How would you aggregate the collected data from all accounts into a single, easily digestible format?
We are creating  .json file with specific field which can be controlled based on the format business wants. Current playbookj will create file in this format disk_utilisation_<Account-ID>_<Region>>.json. we are running task in loop for every child account and for every region. So after running this playbook we will have multiple files in directory "/home/ec2-user/output" (you can change this path).

1. How would your solution scale if the company acquires more companies and AWS accounts in the future?
This is two step process.
First Step: We can use landing zone(control tower) to run a cloudformation template while adding child account to organisation means when parent company will acquire new company and at time of adding it to organisation, a cloudformation will automatically run and create IAM Role which will be assumed by parent. 
Second Step:  Instead of hardcoding child account id, we can run AWS CLI command to get all AWS account Id those are childs, means acquaried later, This is ensure every next run of playbook will also consider new accounts added into orgainsation "list-children". 

this will make not only help while we add new child account , it will also work when we remove child account from organisation, means a company  is no longer part of parent company.