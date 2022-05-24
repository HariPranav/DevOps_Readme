# A DevOps approach to building, Scalable Cloud-Native Data Engineering Applications

![Featured](https://user-images.githubusercontent.com/28874545/170026814-bbc908b9-d83e-4267-b612-36a33a1db017.png)

In the last blog [link](https://medium.com/@haripranav98/building-data-engineering-pipelines-on-aws-5329f3120e77) we created a Flask application on an AWS EC2 instance and built custom data pipelines to interact with Data Engineering tools on AWS. This method of deployment has its own pitfalls, as the entire software development lifecycle which involves the maintenance of code, development of new features, collaboration and versioning is often difficult to handle.
Hence we need to shift to a DevOps approach which helps developers create end to end pipelines from development to testing to production which can deal with collaborative changes.

In this blog post we will be.

1. [Creating a Docker image of the Flask Application](#creating-a-docker-image-of-the-flask-application)
2. [Publishing the image to AWS Container Registry and Running the code in AWS Container Service using AWS Fargate](#publishing-the-image-to-aws-container-registry-and-running-the-code-in-aws-container-service-using-aws-fargate)
3. [Add a Load Balancer for the ECS deployment](#add-a-load-balancer-for-the-ecs-deployment)
4. [Create an AWS Code Commit repository to push code](#create-an-aws-code-commit-repository-to-push-code)
5. [Configure AWS Code Build to build new changes from the Code Commit Repository](#configure-aws-code-build-to-build-new-changes-from-the-code-commit-repository)
6. [Configure Code Pipeline to automatically run steps 1 to 5 once the new commit is made to the Code Commit Repository](#configure-code-pipeline-to-automatically-run-steps-1-to-5-once-the-new-commit-is-made-to-the-code-commit-repository)

# Creating a Docker image of the Flask Application:

We can create a simple flask application which can interact with AWS as shown in the blog post below.

[Using Flask to create custom data pipelines](https://medium.com/@haripranav98/building-data-engineering-pipelines-on-aws-5329f3120e77)

Once we are successful with the creation of the Flask application on Ec2 instance we then need to add a Dockerfile which is used to create a Docker image from the existing Flask application.

1. Create two new files called Dockerfile and requirements.txt in the same parent directory as shown below

![image](https://user-images.githubusercontent.com/28874545/166158371-56af212b-5d56-4498-b833-de7358287de1.png)

2.  Create the requirements.txt file which has all the packages to be installed on the Docker Image.

    $ cd CLOUDAUDIT_DATA_WRANGLER

    $ nano requirements.txt

    Then Add the two package inside the file

        Flask
        awswrangler

Here we don't a specify the version and this ensures that the latest version of the packages get installed.

3.  Add the following lines into the Dockerfile.

        # syntax=docker/dockerfile:1

        FROM python:3.8-slim-buster

        WORKDIR /python-docker

        COPY requirements.txt requirements.txt

        RUN pip3 install -r requirements.txt

        COPY . .

        CMD [ "python3", "-m" , "flask", "run", "--host=0.0.0.0"]

- The first line (# syntax=docker/dockerfile:1) tells what is the syntax to be used while parsing the Dockerfile and the location of the Docker Syntax file

- The second line (FROM python:3.8-slim-buster) allows us to use an already existing base image for Docker.

- The third line (WORKDIR /python-docker) tells docker which directory to use.

- The fourth and fifth lines tells Docker to copy the contents of requirements.txt into a container image's requirements.txt file and then run the pip install command for all the packages and dependencies.

- The sixth line (COPY . .) tells Docker to copy the remainder of the folders to be copied into Dockers container.

- The last line with (-m) indicates to run the Flask app as a module and to make the container accessible to the browser.

## Building the Docker Image:

When we run the command as shown below, it builds a container with the name **python-docker**. This can then be pushed to our Container Registry on AWS.

    $ docker build --tag python-docker .

    $ docker run -d -p 5000:5000 python-docker

Here "-d" will run it in detached mode and "-p" will expose the specific port.

# Publishing the image to AWS Container Registry and Running the code in AWS Container Service using AWS Fargate

We will be using AWS Fargate which is a serverless compute engine for Amazon ECS that runs containers without the headache of managing the infrastructure.

1. Open the AWS console and then search for ECR

   - Click on **Get Started** and then **Create Repository**

   ![image](https://user-images.githubusercontent.com/28874545/166156413-75d5d1f9-d79d-4051-99dd-9c73cd20601d.png)

   ![image](https://user-images.githubusercontent.com/28874545/166156597-46d3d06d-dbb2-49f0-9a55-8c53838feebe.png)

   Here the repository name is flask-app

2. Now click on push commands as shown in the screenshot shown below.

   - ![image](https://user-images.githubusercontent.com/28874545/166156715-39018f0b-65ec-41f1-ade9-1cd09cf1cc22.png)

   We have a list of commands that we need to run on our Ec2 instance which will push the docker image to ECR.

   ![image](https://user-images.githubusercontent.com/28874545/166156818-327d426d-a690-49a5-93cf-bd27c0af5907.png)

   For **Command 1** we need to attach a policy for the Ec2 instance to get the necessary permissions to authenticate with the ECR.

   The links below show how we can attach a policy to the Ec2 machine.

   [Attach IAM to Ec2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html#attach-iam-role)

   [Add ECR execution Policy](https://docs.aws.amazon.com/AmazonECR/latest/userguide/security_iam_id-based-policy-examples.html)

   After we add the permissions we need not run the second command as we have already created the docker image.

   Now for **command 3 and 4** we need to run :

   $ docker tag python-docker:xxxxxxxxxx/flask-app:latest

   _REPLACE THE XXXXXXXX with the statements from your Console_

   $docker push xxxxxxxxx/flask-app:latest

   _REPLACE THE XXXXXXXX with the statements from your Console_

   Here we are tagging the image **python-docker** and then pushing the image to ECR with the name as flask-app.

3. Creating a cluster

   1. Search for ECS on the AWS Console, Click on **create a new cluster**, give a name and choose the default **VPC and the subnets**. Then choose **AWS Fargate** as shown in the image below.

   ![image](https://user-images.githubusercontent.com/28874545/166157117-5e51b5cc-658e-48ae-b807-cca2b991bf76.png)

   2. Then the left pane choose **Task Definition**

   ![image](https://user-images.githubusercontent.com/28874545/166157309-f4608340-27b5-491d-83b7-40462fec9ca8.png)

   In this give a unique name, Enter **flask-app** for the name of the container and the URI which can be found in the ECR screenshot as seen below.

   ![image](https://user-images.githubusercontent.com/28874545/166157388-7fe33fb0-b5c4-4dc5-8b87-c573319e0f53.png)

   For the port mapping choose port **5000 and 80**. Then click on next. For the **Environment** choose Fargate and leave the other defaults as shown below.

   ![image](https://user-images.githubusercontent.com/28874545/166157459-5c17dd33-6cb8-49e1-85fd-86e3ccd4dfc7.png)

   Finally Review and click next. This will add the Container in ECR to Fargate.

4. Running the Task Definitions:

   - As shown in the screenshot below click on on the created Task Definition and click on Run

   ![image](https://user-images.githubusercontent.com/28874545/166157555-b0f06a36-a3d7-44da-bb1a-60677034d0d0.png)

   - Choose the Existing Cluster as shown below

   ![image](https://user-images.githubusercontent.com/28874545/166157616-053ee8b1-41b0-43ee-a609-adc2a4eb14a2.png)

   - Choose the VPC and Subnet, create a new security group and click on deploy as shown below.

   ![image](https://user-images.githubusercontent.com/28874545/166157671-5d6b25e7-858e-4c5a-b0eb-e61dfe521a77.png)

   - Navigate back to **Clusters on the left** then click on the cluster which has been created. Then Click on **Tasks** to get the ENI which has been shown in Black in the image below.

   ![image](https://user-images.githubusercontent.com/28874545/166157869-0d152b0a-8a11-434a-8984-55d230790c7b.png)

   - Click on ENI and also make note of the public IP address as shown below.

   ![image](https://user-images.githubusercontent.com/28874545/166157970-249023fa-900a-4432-a46b-7226e2dc4680.png)

   - Click on security Group and Edit the inbound port rules to allow traffic from Port 5000 and 80 as shown below.

   ![image](https://user-images.githubusercontent.com/28874545/166158122-7239a5d4-b1cd-41d9-8686-9117599a7b31.png)

   ![image](https://user-images.githubusercontent.com/28874545/166158195-963be55c-d609-4f4d-8a45-8fcba4e083bf.png)

   - Once this is done Enter the PUBLIC IP from the ENI and the port 5000

   Eg: **http://publicIP:5000** in the URL and once this is entered YOU HAVE SUCCESFULLY LAUNCHED A DOCKER IMAGE ON THE ELASTIC CONTAINER SERVICE !!

   ![image](https://user-images.githubusercontent.com/28874545/166158258-05d1488a-c735-43e9-8a61-e5188dcce687.png)

# Add a Load Balancer for the ECS deployment:

Open the EC2 Console and then choose the application load balancer as shown below.

![image](https://user-images.githubusercontent.com/28874545/168069807-ca8183ac-031f-492a-b746-f3af741270b8.png)

Make sure to shift to the OLD AWS CONSOLE while doing this step as it will help to follow with the same steps as shown below.

![image](https://user-images.githubusercontent.com/28874545/168070929-81d61586-7a0f-431f-9522-6b157a96f86b.png)

- Choose **Application Load Balancer** and click Create

- Next, Configure it by giving a name and select the VPC and availability zones

![image](https://user-images.githubusercontent.com/28874545/169034353-9ec32f8e-52f1-4b3c-91d9-82567a751e66.png)

- Click Next, select Create a new security group and then click Next

- Give a name to Target group, for Target type select IP and then click Next

![image](https://user-images.githubusercontent.com/28874545/169034491-b28126b8-95f1-40b7-96bc-1e9de4574665.png)

- Click Next, Review it and click Create

- Once created note down the DNS name, which is the public address for the service

# Creating a Fargate Service:

We will use the same task definition to create a **Fargate Service**

- Go to Task Definitions in Amazon ECS, tick the radio button corresponding to the existing Task definition and click Actions and Create Service.

![image](https://user-images.githubusercontent.com/28874545/169035450-c9666d41-07e5-46cf-b898-30bace419d02.png)

- Choose Fargate as launch type, give it a name, do not change the Deployment type (Rolling update), and click Next.

![image](https://user-images.githubusercontent.com/28874545/169035677-85b0e741-fe30-4805-9c27-276f4942382f.png)

Choose the subnets that we have configured in the load balancer

![image](https://user-images.githubusercontent.com/28874545/169035851-d2a6bae6-267f-47d6-9ecc-2e0ca71aa124.png)

Choose Application load balancer for the load balancer type, and then click Add to load balancer

![image](https://user-images.githubusercontent.com/28874545/169036079-e6de31dd-f567-46a0-8b47-0f037c49840b.png)

Select the Target group name that we have created in the Application load balancer and then click Next, Review it and then click Create Service.

- Now we need to configure the Application load balancer security group that we have created earlier, go to the created Application load balancer and click on security groups and then click Edit inbound rules and add a Custom TCP rule with port 5000, as this is the internal port our application is configured in the flask application

- Now, we can check our application is running by visiting the load balancer DNS name in a browser

It will most probably not run and there are very few guides to trouble shoot this issue. The Root cause for this is as follows:

Flow of the application before adding the load balancer:

Client -> URL -> Public IP of Fargate(port 5000 and 80 is opened to a specific IP) -> Response

Once we add the new load balancer and open the ports

Client -> URL -> Load Balancer -> Public IP of Fargate(Only port 5000 and 80 is opened to a specific IP) **(PORT IS NOT OPENED TO THE LOAD BALANCER SECURITY GROUP)**-> Response (As port 5000 is opened for the Security Grp)

Hence this can be seen in the documentation as shown below

![image](https://user-images.githubusercontent.com/28874545/169037040-4ed651b1-712a-4797-bf66-153e42804fe6.png)

To trouble shoot this issue, we need to **OPEN PORT** for the newly created **Security Grp** which is attached to the load balancer.

- Open ECS and select your cluster which contains the newly created service as shown below.

![image](https://user-images.githubusercontent.com/28874545/169038163-4f6b0078-275e-40fa-8c0c-8274a39ed336.png)

- Click on the Task and go to the ENI as shown below

![image](https://user-images.githubusercontent.com/28874545/169038453-6e4157ba-d707-409d-aaec-5824520e1251.png)

- Edit the Inbound rules and add the **Security Grp** which was given to the load-balancer as shown below and also open port 5000

![image](https://user-images.githubusercontent.com/28874545/169038800-8e60281c-27c7-4e8d-b5cc-717e716e04b5.png)

- Now if we open the open the load balancer page and check the DNS address in our browser we should be getting the FLASK APPLICATION.

- Hurray we have successfully deployed the application on ECS and added a load balancer !!

- Another way to know if the application is running successfully is to check the load balancer **Target GROUP**

- Open the Ec2 console, navigate to load balancer and then check the TARGET GRP, if it is showing **Unhealthy or Draining** refer to the links below, it will most probably be the Inbound port rules which have not been opened !. In the screenshot shown below, we can see that it shows that the instance is healthy

![image](https://user-images.githubusercontent.com/28874545/169111085-c0bb3ba7-56ea-4ccf-8a9d-ebe5a2792734.png)

- If we are still getting any 504, or 503 errors refer to the links below:

[aws](https://aws.amazon.com/premiumsupport/knowledge-center/public-load-balancer-private-ec2/)

[aws](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-update-security-groups.html)

[stackoverflow](https://stackoverflow.com/questions/44403982/aws-load-balancer-ec2-health-check-request-timed-out-failure)

# Create an AWS Code Commit repository to push code:

Open the AWS console and search for CodeCommit, click on create a new Repository and give it a name and a **description** as shown in the image below.

![image](https://user-images.githubusercontent.com/28874545/166207560-3cc886c1-d3af-47df-8900-e04e4092e0a3.png)

Add a new file called **buildspec.yml** in the parent directory.

- Now our application folder structure will look as per the screenshot shown below.

![image](https://user-images.githubusercontent.com/28874545/166219011-2809bb37-52cd-401f-8c1a-c8cdacc5acbd.png)

- This file contains the commands which can be used to compile, test and package code.

## ADD A BUILD SPEC FILE and push this repo to git.

![image](https://user-images.githubusercontent.com/28874545/169280328-7df73c84-3de3-45c7-b042-295379a40520.png)

[Download and change according to the image above](https://github.com/dmoonat/sentiment-analysis/tree/master/containerized_webapp)

- We can see that the "image.json" is an artifact file which contains the container name and the image URI of the container pushed to ECR. We then use these in CodePipeline.

- artifacts are created from the container name which is created earlier in EC2 which is **flask-app**

Since this file is a Yaml file we need to check spacing and make sure we follow the right number of tabs and spaces.

- Once the repository is created we need to add specific permission and create Git credentials to access the CodeCommit repository.

- Go to the IAM console, choose **Users** and select which User you want to configure for CodeCommit, and attach **AWSCodeCommitPowerUser** policy from the policies list and Review and then click Add Permission.

![image](https://user-images.githubusercontent.com/28874545/166207923-4b408bb7-5710-4fe3-af0d-2073a60751d3.png)

- Configure Git using the blog post below:

[Configure Git Locally](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-gc.html?icmpid=docs_acc_console_connect_np)

- Follow the documentation below to Add Git Credentials for the AWS account. Once this is added, open the repository created in Code Commit and then run the following commands.

CD into your repository and run the commands as shown

- Add the files in your new local repository. This stages them for the first commit.

$ git init

- Commit the files that you've staged in your local repository.

$ git add .

$ git commit -m "First commit"

At the top of your GitHub repository's Quick Setup page, click to copy the remote repository URL.

In the Command prompt, add the URL for the remote repository where your local repository will be pushed.

- Sets the new remote

$ git remote add origin <remote repository URL>

Refer to the screenshot below to find the URL:

![image](https://user-images.githubusercontent.com/28874545/169132983-5eaf5efc-176b-41bb-9d99-063bb4e00a38.png)

- Verifies the new remote URL

$ git remote -v

- Push the changes to the **master** branch

$ git push origin master

# Configure Code Pipeline to automatically run steps 1 to 5 once the new commit is made to the Code Commit Repository:

We can then use Code Pipeline to configure builds from CodeCommit to the ECR which will inturn run the image on ECS.

1.Go to **CodePipeline** and click on **GetStarted**

Choose the **Service**, **Repo** and the Click on Next. Select the Repo name and Create a new service Role as shown below

![image](https://user-images.githubusercontent.com/28874545/166220689-93dbea15-c080-40a8-949a-89659ede2a98.png)

2.Choose the Source as **AWS code Commit** and Repo name from the dropdown. Then Choose the **Master branch** and click on Next

![image](https://user-images.githubusercontent.com/28874545/166220639-a5a874bd-2ac1-4872-8418-6b829b8e1620.png)

3.Next, for the build provider select CodeBuild and create a new build project, give it a name and configure it as follows

![image](https://user-images.githubusercontent.com/28874545/169281262-62402be6-6322-4c16-893e-93538956204e.png)

Make sure to **Check the box: Privileged** and make sure that the new service role is created and the Ec2 permission is added as shown below.

![image](https://user-images.githubusercontent.com/28874545/169281298-a2a82932-1d25-4c24-8cc1-b71ca7ad2608.png)

4.It will create a New service role, to which we have to add ECRContainerBuilds permission. For that, open the IAM console in a new tab, go to Roles and search and select the above-created role, and click Attach policies. Search for ECR and select the policy as below and click Attach policy

![image](https://user-images.githubusercontent.com/28874545/169281388-037ac079-c521-4970-926c-4e2b6709ee69.png)

For Deploy provider, select Amazon ECS, cluster, and service name. Also, add the name of the image definitions file as ‘images.json’ that we will create during the build process

Here choose the name from the dropdown and ignore the names from the images below.

![image](https://user-images.githubusercontent.com/28874545/169281469-ce35dbdc-42ef-43c6-b6b4-84406bc43ecb.png)

Once all the configurations are you should have something like this:

![image](https://user-images.githubusercontent.com/28874545/169282910-3e184823-be0b-448b-b7b8-328c094e9736.png)

You can then make changes in the code push the commit to the repository and the pipeline will run and deploy the Latest changes to ECR via the DevOps Pipeline.

If you do, You **yyess, You** have implemented have implemented a cloud native DevOps scalable pipeline for Machine Learning and DataEngineering .

**Don't forget to leave a Like, Share and Comment !!!!!**

References:

[Analytics Vidya](https://www.analyticsvidhya.com/blog/2021/07/a-step-by-step-guide-to-create-a-ci-cd-pipeline-with-aws-services/)

[AWS Docker](https://docs.aws.amazon.com/codebuild/latest/userguide/sample-docker.html)

[Code Build](https://docs.aws.amazon.com/codebuild/latest/userguide/change-project-console.html)

[Code Pipeline](https://docs.aws.amazon.com/codepipeline/latest/userguide/welcome.html)
