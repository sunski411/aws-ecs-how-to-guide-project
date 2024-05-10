**(Ritual Roast Web APP) AWS Amazon ECS and AWS Fargate - Step-by-Step Tutorial**

In this project, AWS Elastic Container Service (ECS) is discussed as a platform for deploying and managing containers. We will also demonstrate how to build and deploy a complete ECS architecture, including a multi-tier application for high availability and fault tolerance.

This application is a multi-tier application, so it will have:

a) Frontend UI (HTML, JavaScript, beautiful images, and a marketing strategy).

b) Customer form to engage with the customers and gather their information for marketing.

c) This website will connect to a backend DB.

d) DB CRUD Operations.

Containers and virtualization both provide computing services but differ in how they achieve virtualization. A virtual machine is a digital copy of a physical machine (complete with its own guest OS, systems files, and libraries). VMs also allow you to run your applications in isolation on the same physical hardware using its own guest OS, runtime, and libraries.

Containers are like VMs in that you can run multiple containers on a single hardware, however, a VM has the host OS, hypervisor, and VMs, each of which has its own guest OS. With containers, you still have the host hardware (AWS Host Hardware), it also has a host OS (this could be running Windows, Linux environments, etc.), also a container engine (Docker) running as isolated processes on the host OS. When deploying containers, we have a smaller footprint of resource utilization to run an application efficiently.

Containers are created from a container image (Docker image). We start with a Dockerfile (a manifest that describes the various components of the image). After building the image, you can go ahead and launch and run the container from the image.

Amazon Elastic Service (ECS) orchestrates and runs highly secure, reliable, and scalable containers. You can use the ECS service as a managed service from AWS (Fargate) or if you need to manage your own containers and the underlying host machines (EC2 instances), then you can choose the EC2 launch type.

**Components of ECS**

When you work with ECS, you still need to build a container image. To build that image, you will need to use a Dockerfile. This Dockerfile will then architect your image which will then be placed in a repository (Docker Hub or Amazon Elastic Container Registry ECR).

The ECR provides a place to create repositories to store your Docker images. The best thing about ECR is that it integrates seamlessly well with other AWS services.

Once the container image has been pushed to the repository, we can start using the various ECS components to deploy the containers.

**Task definition** is a blueprint for your application. It’s a text file in JSON format that describes the parameters of one or more containers to form the application. It will contain several parameters such as the image, the IAM roles the task will be using for the container application.

**Container definition** is used to describe the different containers that are launched as part of the task (container).

Before you deploy the task, you must create an ECS cluster (a logical grouping of the containers that runs on infrastructure on AWS platform).

**ECS service** is created to manage the scalability of our task (specify the number of tasks to run).

**Project**

1) First thing we need to do is build out our 2-tier VPC architecture because the compute services and database services will all be hosted within a VPC architecture. For best practice, we will be working in at least two availability zones for high availability and fault tolerance.

a) Build VPC (This is a Virtual Private Network you build within your AWS account to house all the resources for a specific project). In accordance with best practices, you want to separate your different subnets between the public and private environments of the VPC and deploy the solution across multiple availability zones for high availability.

b) Create an Internet gateway to allow internet traffic into the VPC and attach it to our VPC.

c) Create 6 subnets and spread both to availability zones (2 private subnets 1a-1b, 2 private app subnets 1a-1b, and 2 private data subnets 1a-1b).

d) Create a public Route table to route traffic to public subnet from the Internet gateway and associate it with both public subnets 1a and 1b.

e) Create a Nat gateway that will allow traffic into the private subnets from the private Route table.

f) Create a private Route table to route traffic into the private subnet through the Nat gateway and associate it with the private app and data subnets.

2) The next thing we need to do is build our Security groups for controlled traffic into our VPC.

a) Create the load balancer security group. This will allow inbound traffic from the internet from port 80 and 443 (HTTP and HTTPS) with a source from 0.0.0.0/0.

b) Create the app security group. This will only allow inbound traffic from the application load balancer security group, and we will open port 80 and 443.

c) Create the database security group. This will only allow inbound traffic from the app security group, and we will open port 3306.

d) Create an EC2 instance endpoint connect security group. This security group will only allow outbound traffic to port 22 from the VPC CIDR.

e) Create an SSH security group. This will only allow inbound traffic from the EC2 security group, and we will open port 22.

3) Create a repository in Amazon ECR to host our container image (Docker image) that will then be pulled down by the ECS service to launch our container. We can create our ECR using AWS CLI. To use AWS CLI to create the ECR repository, let us open our PowerShell and run this command below:

```
aws ecr create-repository --repository-name <repository_name> --region <region>
```

Ensure you make a note of the image URI because we will be using it later in this project.

4) Next, we will build our Docker image. To build a Docker image, we need a Dockerfile (provided in my GitHub repository). While using your local machine to interact with AWS, ensure AWS CLI is installed and configured with an IAM user that has sufficient privileges to interact with the AWS environment (Administrative Access) and configure the AWS CLI with these privileges.

We will be deploying an EC2 instance to use as our Docker build server. This EC2 instance will be needing some permissions to upload our Docker image from the build server into our Amazon ECR.

5) Create permissions and roles. We will use some pre-defined roles created by AWS to avoid complexity.

a) In the IAM page, select “Role”.

b) For Select Trusted Entity, select "AWS Service”.

c) For Use Case, select “EC2”.

d) Click on “Next”, and on the next page, search for “AmazonSSMManageInstanceCore”. This policy allows you to remotely connect to a server using the Session Manager.

e) We will also attach “EC2InstanceProfileForImageBuilderECRContainerBuilds”. This policy allows us to upload ECR images. Click, give role a name, and create role.

6) Next, let us create our EC2 and give it a name:

a) We will use Amazon Linux 202

3 AMI for our OS.

b) Instance type will be t2.micro (free tier).

c) Since we are connecting to our server using EC2 instance connect endpoint, we can proceed without “Key pair”.

d) Edit the Network settings and select our “configured VPC for the project”.

e) Our instance will reside in any private app subnet.

f) Select the app security group for our instance.

g) Scroll down to “Advanced Details”, click on the dropdown, and select the role we created for the “IAM instance policy” and create the EC2.

7) SSH into the server by running the command below:

```
aws ec2-instance-connect ssh --instance-id <instance-id>
```

a) This will log us onto our server. Change to root user and update the server to check for any updates:

```
yum update -y
```

b) To build our Docker image, we must install Docker on our server first:

```
yum install docker
```

c) Start the Docker service:

```
service docker start
```

d) Add the ec2-user to the docker group so that you can run Docker commands without using sudo:

```
sudo usermod -a -G docker ec2-user
```

e) Log out and log back in to allow these changes to take effect.

8) To build our Docker image, we need to get access to our application code in our GitHub repository.

a) Select the application code in the GitHub repository and right-click on the “raw” tab to copy the link address.

b) Return to our Bash shell and type:

```
wget <paste the link address>
```

c) Unzip the zip folder to extract the contents of the folder:

```
unzip <folder.zip>
```

d) Change directory to the unzipped folder. Here you will find the contents of the folder including the Dockerfile which we will use to build the Docker image, also the index.html file which is the front-end of our application.

e) Navigate to our ECR repository and copy the image URI, we will be using this to build our image. To build the Docker image, run this command:

```
docker build -t <image_uri> .
```

9) Now that we have built our Docker image with the Docker file, the next thing we would do is push the image to the ECR repository.

a) Go to the container registry where we create the image, click on the repository name, and click “view push commands”. Here, you will find the commands to push the container image to our ECR repository. For this project, we will run only command (1) and (4).

10) Next, we will create our RDS MySQL database to host all the data that was collected from the front-end form.

a) Create the subnet groups to deploy the MySQL database.

b) Create the MySQL database using the latest version. In a real-life scenario, we will deploy a standby instance (so we’ll have a master’s instance and a standby instance) and then configure multi-AZ to ensure high availability and failover ability for the database architecture.

11) Configure Secrets manager, so that we can get the credentials of the database managed by the Secrets Manager and implement credentials rotation. We are also using SM to dynamically retrieve database credentials in a secure manner. When SM is deployed, the purpose of it is to ensure that access to the databases is done as securely as possible. The application needs to authenticate itself against the database. How this works is simply creating the SM, pointing it to the database and specifying the credentials. SM automatically can rotate the database credentials with the use of Lambda functions. When you configure a rotation for SM, it invokes a CloudFormation template in the background which builds a Lambda function that takes responsibility for rotating the secrets at regular intervals. The Lambda function needs to talk to the database and SM. To do so, it needs to be associated with a security group that allows communication with the database.

a) Create a Lambda security group opening port 3306 to allow traffic from the database security group. Or you can setup VPC endpoint to interact with SM.

b) Create and configure Secrets Manager, so we can get the credentials of the database managed by SM and we can implement rotation of those credentials.

i) Select “Credentials for Amazon RDS database”.

ii) Provide user credentials (username & password).

iii) Select the DB that was created.

iv) Provide secret name and description and click next.

v) Enable rotation (this is where CloudFormation will go in and create the Lambda function).

vi) Set up rotation schedule to your preference.

vii) Enable “create a rotation function and give the Lambda function a name.

viii) Review your configurations and store the secrets.

c) CloudFormation will create the RDS secrets manager secrets rotation and create a Lambda function.

12) Create an Elastic Load Balancer (Application Load Balancer) to distribute loads evenly to our containers. This will help improve the application performance by increasing response time and reducing network latency.

a) First create the Target groups. ECS Fargate requires “IP” as the target type.

b) Give your Target group a name.

c) The Protocol: Port is “HTTP”.

d) Select your IP address type, in this project is IPv4.

e) Select your VPC and choose “HTTP1” for the Protocol version.

f) This application code has the index.html file but it also has a Health check file that is being used to monitor the availability of the website. Where it says, “Health check path”, type /health.html. There’s a file called “health.html that performs all the health checks. If this file is accessible and responding, the load balancer will mark the ECS services as healthy.

g) In the next page, you will register the targets. The network should be the VPC where our application is hosted. Because we haven’t set up our containers, there won’t be any targets yet. We will reference the target group when we’re setting up our containers (so leave “Review targets” empty) and click create.

h) Now that our Target group has been configured, we can deploy our Load Balancer.

i) Go to Load Balancer and click on “Create”.

j) Give the Load Balancer a name.

k) The scheme should be “Internet-facing”, this will accept traffic from the web and distribute to our targets.

l) For the “Network mapping”, select the right VPC.

m) Choose the Az’s where the subnet resides. The ALB are deployed in the public subnets across both AZs to receive traffic from the internet.

n) Select your Security group for the ALB to accept inbound traffic.

o) When the ALB receives traffic from the internet, it sends it to the Listener. Select the TG we create we earlier. Review and create ALB.

13) Create IAM Roles and Policies for ECS Task. The ECS needs permission to download the image from the ECR. It will also need permission to send log information to the CloudWatch. The container needs permission to reference the secrets in SM to talk to the database.

a) In the IAM dashboard, click create a policy to create a customer managed policy.

i) Search for “Secret manager” policy.

ii) Under Secret manager, in “Actions allowed” search for “GetSecretValue”.

iii) Under “Resources”, you want to be specific on the resource. Click “Add ARNs”

 to add the ARN of the secret.

iv) Retrieve the secrets ARN from the SM and paste it in the resource ARN of our policy and click “Add ARN”.

v) Provide a name for the policy and create the policy.

b) Next, we will create a Role. This role will have 2 policies attached to it: (i) the secret policy we just created, (ii) policy to interact with the ECR and CloudWatch.

i) Click on “Create role” and select “AWS Service” for Trusted entity type.

ii) The use case is for Elastic Container Service and select Elastic Container Service Task and click next.

iii) On this page, we will attach an AWS managed policy. Search for “AmazonECSTaskExecutionRolePolicy” and check the box (you can expand the policy to see what permissions are available).

iv) We will add the policy we created to this role. Search for the policy name in the search bar and check the box.

v) Give the role a name and create a role.

14) Create ECS Cluster. ECS Cluster is the logical grouping of our containers (either Fargate or EC2 selection of containers).

a) Navigate to ECS, click on “Cluster” and “Create cluster”. Give the cluster a name.

b) Select “AWS Fargate (serverless) as the infrastructure type and click “create”.

15) Create the ECS task definition. The ECS task definition requires several parameters to build the container. For example, we will provide information on the AWS SM we will be using to obtain the credentials of the database. We will also define necessary IAM roles, configure environmental variables (e.g. the name of the secret in SM) and include which Docker image to use.

a) Click on “Task definition” and create task.

b) For “Infrastructure requirements” we are using AWS Fargate.

c) OS Architecture is “Linux/X86_64” for Windows machine.

d) Because we are using Fargate, the network mode is automatically set to “awsvpc”.

e) Specify the task size (how many CPU’s and memory to reserve for the task).

f) Select the role we created for the “Task role” and “Task Execution Role”. This has permission to allow the container agents to make API calls on your behalf. The Task role will use the permission allocated to the role to access the SM.

g) Next, let’s specify the container name and container image we will be using. For the image URI, we can copy the URI from the image we pushed to our ECR repository.

h) Define the “Port mappings”:

i) Container port – 80

ii) Protocol – TCP

iii) App protocol -HTTP

i) Next, let’s define the environment variables:

i) Key: DB_SERVER

ii) Value type: set as “value”.

iii) Value: < RDS_endpoint> and click on the environment variable tag again at the bottom:

iv) Key: DB_DATABASE

v) Value: set as “value”

vi) Value type: <database_name> and create another environment variable:

vii) Key: SECRET_NAME

viii) Value type: set as “value”.

ix) Value: <secret_name> and finally, create environment variable for specify the region:

x) Key: AWS_REGION

xi) Value type: set as “value”.

xii) Value: <region>

j) Leave all other configurations as they are and click “Create” to create the Task definition.

16) Create our ECS Service which will deploy our container into the Cluster, and then we can access our application.

a) Select the cluster and in the cluster, click on the “service” tab and click “create”.

b) For the “compute configuration (compute options)”, leave it as “Capacity provider strategy”.

c) “Capacity provider strategy” leave it as “Use custom (Advanced)

d) The “Capacity provider” is Fargate.

e) “Platform version” is LATEST.

f) “Application type” is Service.

g) Select the Task definition we created for the “Family”, and we are using the LATEST revision.

h) Give the Service a name.

i) The “Service type” is set to Replica.

j) Let us go with 1 “Desired task”.

k) In the “Networking” expand it and select the VPC our application is running.

l) Next, select the “app subnets.” We are deploying our containers in private subnets for enhanced security.

m) Allocate the containers to our “app security group” because it has the right set of inbound rules.

n) Turn off the “Public IP” tab, we do not need it.

o) Expand the Load Balancing tab. Select the Load balancer allocated to this project.

p) Use the existing Listener.

q) Use the existing Target group.

r) Next, we will configure the Service Auto Scaling. This is where the task is defined on how to scale in or out.

i) Check the box “Use service auto scaling.”

ii) Scaling policy type is “Target tracking.”

iii) Give the ASG a policy name.

iv) For ECS service metrics, select “ECSServive AverageCPUUtilization.”

v) Target value = 70

vi) Scale-out cooldown period = 300

vii) Scale-in cooldown period = 300

viii) Scroll down and click “create” to create our ECS service.
