# Cloud Journey
My notes on learning Cloud Computing

A good reminder: https://youtu.be/U1-g2qlnTO8
This one too: https://youtu.be/vWDhQNXM4K8
Okay, got it!: https://youtu.be/gaCY4QxfSzA
Always to remember: https://youtu.be/Rqinrxtt1fg


Learning IaC from Terraform, I now understand what they mean by Declarative.
With code, it's imperative, meanining we are instructing it to get a result. Here, in declarative, you just give it the result you want. i.e. instead of wanting 5 EC2 servers from already having 2 deployed by saying 'Add 3 more EC2 servers', we just say, 'Deploy 5 EC2 servers.' With IaC, it overrides everything (the 2 EC2 already in operation), and you get 5 EC2 servers.


Finished the Learn to Code 18 Linux CTF. A good refresher on Linux commands. Onwards! Right now, I will focus on breaking and making mistakes fast first before learning optimizing.

<img width="967" height="975" alt="awww" src="https://github.com/user-attachments/assets/8fe153d7-8f77-460e-9d0c-e832355ad436" />


Finished 4 tickets on the AWS cloud!
I have learnt the inner workings of CloudWatch alarms on error rates, synthetic checks that ping external endpoints every minute and alert if they fail.
I have also learnt how to troubleshoot and connect all Networking basics in a 3-tier VPC.
I lastly learnt how to use subnets checkers, terraform and bash scripts automation to ensure correct networks and allow only the proper security groups or singular IPs.

<img width="1032" height="765" alt="image" src="https://github.com/user-attachments/assets/1f9089c6-b916-4135-bcd2-e83fc75dc4e8" />


Now to run an n8n server to test out some automations using Elastic IPs. Woah!


Just came across my first pseudo-optimization!
So, running multiple n8n workflows on a 1GB RAM t3.micro was something I did not know was a strecth on resources. Lesson well learnt as I had to check using docker ps to compare container uptimes and to see I had only ~118MB of free room to begin with! n8n's process choked, crashed, and Docker automatically restarted it (that's what "restart: always" in the compose file does). But since the memory pressure never actually went away, it crashed again. And again. That's the loop I saw in the logs = "Initializing → ready → crashed → Initializing → ready → crashed," over and over and over and over and you get it.
The fix: adding a 2GB swap file. It'll give n8n the headroom to survive memory spikes without crashing. But a swap is a bandage, not a permanent cure.

2 things I hate: wasting money and being unsafe. 
I was unsafe: I got my accessKeys leaked. 
A good lesson on how to remove IAM access and quickly before a hacker drained my credits:
```bash
:~$ aws s3 ls
2026-06-30 15:05:23 nana-iam-test-1782826819
:~$ aws s3 ls s3://nana-iam-test-1782826819 --recursive
2026-06-30 15:05:43         54 test-file.txt
:~$ aws s3api get-bucket-policy --bucket nana-iam-test-1782826819 2>&1
```


An error occurred (NoSuchBucketPolicy) when calling the GetBucketPolicy operation: The bucket policy does not exist
```bash
:~$ aws s3api get-public-access-block --bucket nana-iam-test-1782826819 2>&1
{
    "PublicAccessBlockConfiguration": {
        "BlockPublicAcls": true,
        "IgnorePublicAcls": true,
        "BlockPublicPolicy": true,
        "RestrictPublicBuckets": true
    }
}
:~$ aws s3 rm s3://nana-iam-test-1782826819 --recursive
aws s3 rb s3://nana-iam-test-1782826819
delete: s3://nana-iam-test-1782826819/test-file.txt
remove_bucket: nana-iam-test-1782826819
:~$
```
Lesson learnt!

I am putting off journal api for now to go through Ghana's OneMillionCoders' course on cloud practitioner. This is so helpful, because yes, I will admit, I didn't know what an S3 bucket really was. Now, I know forever. I just finished the Learn To Cloud section of making a bucket, adding IAM users and special accesses and deleting everything.



Just wrapped up the creation of the Python, FastAPI and PostgreSQL capstone project I will be working on going forward: a journal API with full CRUD endpoints, Pydantic input validation, structured logging and an OpenAI SDK AI-powered analyzer. I had also gotten an early start on using some GitHub Actions.
I especially like the AI addition, not just because I love AI, but that we are getting the most requested skills in real time instead of material from years ago. AI integration didn't even exist roughly two years ago, and now we have it in almost everything! I also debugged devcontainer failures, git commit workflows and static type checked with Pyright. All 50 tests passing with flying green colours! Gwyneth Peña-Siguenza, your vast knowledge on what an engineer needs to know tomorrow truly shines through and I hope other fellow members are enjoying the course just as much as me.
I will still need to learn more on Pyright/mypy to make more quality, maintainable code. Also, because my laptop is currently full with university projects, I will be learning the cloud on the cloud. Thank you Github Codespaces! I will redo this whole course until everything becomes second nature.
