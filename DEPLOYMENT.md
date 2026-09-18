# Deployment Evidence

Fill this in as you go. Paste real output, not descriptions of output. A TA reads this
file with you at recitation.

## 1. Deployed URL and instance id

<!-- The ServiceUrl and InstanceId outputs. Paste both here every time
describe-stacks prints them, for the healthy deploy and for scenario 2. Both
change on every recreate, and you will need them for curls and sessions. -->

-------------------------------------------------------------------------
|                            DescribeStacks                             |
+------------+----------------------------------------------------------+
|  InstanceId|  i-0581d743bf18a9cb7                                     |
|  ServiceUrl|  http://ec2-34-207-111-96.compute-1.amazonaws.com:8080   |
+------------+----------------------------------------------------------+

------------------------------------------------------------------------
|                            DescribeStacks                            |
+------------+---------------------------------------------------------+
|  InstanceId|  i-0a81b7f60cf5dc208                                    |
|  ServiceUrl|  http://ec2-34-230-5-223.compute-1.amazonaws.com:8080   |
+------------+---------------------------------------------------------+

-------------------------------------------------------------------------
|                            DescribeStacks                             |
+------------+----------------------------------------------------------+
|  InstanceId|  i-06ccbfb649daaa8e9                                     |
|  ServiceUrl|  http://ec2-98-84-151-150.compute-1.amazonaws.com:8080   |
+------------+----------------------------------------------------------+

## 2. External health check

Run the check from your own machine, not from the instance. Paste the command and the
response.

```
$ curl http://ec2-34-207-111-96.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}%   
```

## 3. What the template created

Three or four sentences, your own words. What compute, what network access, and what
glue made the service start.

The CloudFormation template created a `t3.micro` EC2 instance running Amazon
Linux 2023 and attached a security group to it. The security group allows
public TCP traffic on the configured service port (8080) and also opens port 22
as a fallback for SSH access. The instance's `UserData` installs and starts
Docker, pulls the public `lab04-service` image, and runs the container with the
configured port mapping and `PORT` environment variable. The template also
attaches the Learner Lab instance profile for SSM access and outputs the public
service URL and instance ID.

## 4. Scenario 2 diagnosis

**The failing curl** (command and output):

```
$ curl http://ec2-34-230-5-223.compute-1.amazonaws.com:8080/api/health
curl: (28) Failed to connect to ec2-34-230-5-223.compute-1.amazonaws.com port 8080 after 75030 ms: Couldn't connect to server
```

**The log line that told you what was wrong:**

```
$ sudo docker ps
CONTAINER ID   IMAGE                                     COMMAND                  CREATED         STATUS         PORTS                                       NAMES
5495f81acaa1   ghcr.io/cmu-17-214/lab04-service:latest   "/__cacert_entrypoin…"   5 minutes ago   Up 5 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   lab04-service

$ sudo docker logs lab04-service
lab04-service listening on 9090
```

**What was wrong, and the fix you applied:**

Docker forwarded host port 8080 to container port 8080, but the
`PortOverride` parameter set the application's `PORT` environment variable to
9090. The log line `lab04-service listening on 9090` confirmed that no service
was listening on container port 8080. I fixed the issue by deleting the broken
stack and recreating it with `infra/params-healthy.json`, which leaves
`PortOverride` empty and makes the application listen on port 8080.

**The healthy curl after the fix:**

```
$ curl http://ec2-98-84-151-150.compute-1.amazonaws.com:8080/api/health
{"status":"ok"}% 
```

## 5. Teardown proof

Paste the delete output, or describe the console evidence that the resources are gone.

```
$ aws cloudformation describe-stacks \
  --stack-name lab04-service

aws: [ERROR]: An error occurred (ValidationError) when calling the DescribeStacks operation: Stack with id lab04-service does not exist
```
