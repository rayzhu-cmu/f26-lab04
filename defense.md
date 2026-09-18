# Milestone 1

Show your TA: the URL, the curl output, and your what-got-created summary.


I deployed the container to AWS using a CloudFormation template and the healthy parameter file. The template created a t3.micro EC2 instance running Amazon Linux 2023 and attached a security group to it. The security group allows public TCP traffic on port 8080 so the service can be reached externally, and it also opens port 22 as an SSH fallback. The instance's UserData installs and starts Docker, pulls the public service image, and starts the container with port 8080 mapped from the host to the container. The template also attaches LabInstanceProfile for SSM access and outputs the public service URL and instance ID.

I waited for the stack to reach CREATE_COMPLETE and then allowed additional time for the EC2 UserData to install Docker and start the container. From my own machine, I sent a request to the /api/health endpoint. The service returned {"status":"ok"}, which confirmed that the EC2 instance, security group, Docker port mapping, and application were all working together.

# Milestone 2

Show your TA: the failing curl, the log line, your diagnosis, and the healthy redeploy.


In Scenario 2, the external health check timed out on port 8080. I waited long enough to rule out normal EC2 initialization time, so this was a persistent deployment failure rather than an early request. The docker ps output showed that Docker was forwarding host port 8080 to container port 8080, but the application log said lab04-service listening on 9090.

# Milestone 3

Show your TA: the teardown proof.


The security group allowed incoming traffic on port 8080, and Docker forwarded host port 8080 to container port 8080. However, the Scenario 2 parameter file set PortOverride to 9090. This value was passed to the container as the PORT environment variable, so the Java service listened on port 9090 instead of port 8080. Therefore, requests reached container port 8080, but no process was listening there.

I fixed it through the infrastructure configuration rather than manually changing the running container. I deleted the broken CloudFormation stack and recreated it using infra/params-healthy.json. In the healthy parameter file, PortOverride is empty, so the template uses ServicePort, which is 8080, as the container's PORT value. After the replacement deployment finished, the new /api/health request returned {"status":"ok"}.