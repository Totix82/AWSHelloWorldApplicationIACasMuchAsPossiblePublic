# AWSHelloWorldApplicationIACasMuchAsPossiblePublic

Sourced from https://github.com/SumindaD/Fargate-ECSCluster-Cloudformation, credits to SumindaD

Instead of manually creating an AWS CodePipeline, an ECS Fargate Cluster infrastructure and configuring a CI/CD Pipeline with Github, Let’s configure a Cloudformation template to automate all of that for us.
Following AWS services will be deployed.

Cloudformation
CodeBuild
CodeDeploy
CodePipeline
VPC with 2 Subnets
ECR
Application Load Balancer
Internet Gateway
Fargate ECS Cluster
A sample Docker container running NodeJS hello world app.

It’s not possible to create all of these AWS Resources using a single Cloudformation template simply because in order to create an ECS Cluster, the Docker image needs to be in the ECR.
Running all of these creations in a single Cloudformation template would result in a failure during the ECS cluster creation simply because there would be no Docker image available in the ECR as the ECR itself is also being created by the same Cloudformation template.

Therefore the idea is to use two Stacks: one to deploy only the CI/CD Pipeline using a Cloudformation template and once it succeeds on creating the Docker image, deploy the next Cloudformation template containing the Fargate ECS Cluster infrastructure using this pipeline itself.

There are 3 stages in AWS CodePipeline.
1. Source Stage
Github as source
2. Build Stage
Build provider will be AWS CodeBuild. It will be used to build the Docker image, to push the Docker image to the ECR and to Output Docker ECR image URL to a JSON file saved in a S3 bucket so the deploy stage can use it.
3. Deploy Stage
Deploy provider will be Cloudformation in this case. This stage will receive the source code pulled from Github as an input artifact which contains the Cloudformation template it should deploy. This template contains the complete Fargate ECS Cluster with a VPC, 2 Subnets and a Load balancer.
Parameters that are required (ECR Image URL from the build stage) for this Cloudformation template will also get passed into the template at this stage.


**** Deployment process: ****

User manually deploys the initial Cloudformation template containing the CodePipeline infrastructure (CodePipeline.yml).
Cloudformation starts the deployment of the AWS CodePipeline infrastructure.
After the Cloudformation deployment, CodePipeline pulls the source code from Github in the source stage.
CodePipeline builds the docker image and pushes to the ECR in the build stage using AWS CodeBuild service.
CodePipeline starts the deployment of the Cloudformation template (Fargate-Cluster.yml) containing Fargate ECS Cluster in the deploy stage.
To summarise, there will be 2 Cloudformation template deployments. The first one will be done by the user which will create an AWS CodePipeline and then the second one will be done by the AWS CodePipeline itself.





AWS CodePipeline Infrastructure in Cloudformation (CodePipeline.yml):

Following resources will be created from CodePipeline.yml template.
ECR Repository
S3 Bucket
IAM Role for CodePipeline Execution
IAM Role for CodeBuild Execution
IAM Role for Cloudformation Execution
AWS BuildProject
AWS CodePipeline
ECR Repository

Things to notice:

a) In the AWS CodeBuild part of the template note that 
ECR location to push the build image is passed via EnvironmentVariables property as ECR_REPOSITORY_URI.
ServiceRole property provides an AWS Role with permissions for AWS CodeBuild service to manipulate AWS resources. This is also defined in the same Cloudformation template as below and referenced in the ServiceRole property in AWS CodeBuild template.

Source -> Type is also set to CODEPIPELINE in order to let the AWS CodePipeline service to manage artifacts.
Source -> BuildSpec is set to value buildspec.yml. This file resides in the Github repository with commands to build and push the docker image to the ECR as displayed below.

Note that after Docker image is built and pushed to ECR, artifacts -> files specifies an output artifact file called imageDetail.json. This file gets created by the following command in the buildspec.yml file.
printf ‘{“ImageURI”:”%s:%s”}’ $ECR_REPOSITORY_URI $IMAGE_TAG > imageDetail.json
This command outputs the ECR image repository url to the imageDetail.json file in order for the Cloudformation to use it during the ECS Cluster deployment.

b) In the AWS CodePipeline part of the template note that
ArtifactStore -> Location specifies the S3 bucket name defined in the Cloudformation template as the artifact location. CodePipeline will use that S3 bucket to store the downloaded source code from Github and build artifacts from CodeBuild service.
RoleArn property specifies the AWS role that gives permission to AWS CodePipeline service to manipulate AWS resources. This is also defined in the Cloudformation template itself as displayed below.

c) In the AWS CodePipeline Source Stage part of the template note that
GitHub specifies the provider as Github and Configuration section defines Repo, Branch, Owner and OAuthToken. 

Owner of the Github repo needs to generate a token and specify here in order for AWS CodeBuild service to have permission to pull the source code. Refer to this Github 
https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token on how to do that.

OutputArtifacts property specifies the path for the source code to be downloaded in the S3 bucket that was configured in ArtifactStore -> Location.

d) In the AWS Build Stage part of the template note that
Provider: CodeBuild specifies this stage will be using the AWS CodeBuild that was configured earlier in “AWS CodeBuild” section. This stage takes the output artifacts that was specified as OutputArtifacts in source stage as an input in InputArtifacts property.
During the execution of this stage, AWS CodePipeline will be running the AWS CodeBuild. Build commands specified in the buildspec.yml file will be executed by AWS CodeBuild to build, push the Docker image to ECR and finally write the ECR image URL to imageDetail.json and output it to S3 bucket as a Build Artifact.

e) In the AWS Deploy Stage part of the template note that
this is where the complete infrastructure for the Fargate ECS Cluster will be deployed. Including VPC, Subnets, Load Balancers, Internet Gateway, Routing Tables etc.
Provider: CloudFormation specifies that this deployment is done via Cloudformation.
InputArtifacts are provided from both Source and Build stage.
Source stage provides the source-output-artifacts which contains the Cloudformation template (Fargate-Cluster.yml) to be deployed.
Build stage provides the build-output-artifacts which contains the ECR Image URI inside the imageDetail.json file.
ActionMode: CREATE_UPDATE specifies that Cloudformation will be creating or updating resources.
Capabilities: CAPABILITY_NAMED_IAM allows Cloudformation to create IAM roles defined in the template itself.
ParameterOverrides section passes parameter values from AWS CodePipeline (CodePipeline.yml) to the Cloudformation deployment template (Fargate-Cluster.yml) such as ImageURI, Stage and ContainerPort. All of these are defined as parameters in the Fargate Cluster Cloudformation template as well which you’ll see below.

Please note that ImageURI is extracted from the imageDetail.json file using the function Fn::GetParam.





AWS Fargate ECS Cluster Infrastructure in Cloudformation (Fargate-Cluster.yml):

Following are the resources that will be deployed for the Fargate ECS Cluster. 
ECS Cluster
VPC
2 Subnets
RouteTable
Route
2 SubnetRouteTableAssociations
InternetGateway
VPCGatewayAttachment
IAM Role for ECS Task Execution
ECS TaskDefinition
2 SecurityGroups
LoadBalancer
TargetGroup
Listener
ECS Service


a) In the ECS service part of the template note that

DependsOn: LoadBalancerListener configuration lets Cloudformation know that this resource needs to be created only after the LoadBalancer is created. You can specify dependencies this way in Cloudformation.
DesiredCount: 2 specifies there should be 2 instances running.
NetworkConfiguration -> Subnets specifies the 2 subnets the instances should be running. In this case 1 instance will be running in both subnets adhering to the rule of DesiredCount: 2.
LoadBalancers -> ContainerPort defines the port of the container. Note that the value for this parameter is passed by the AWS CodePipeline’s Deploy stage as discussed in previous section.
LoadBalancers -> TargetGroupArn points to the TargetGroup of the LoadBalancer that will be used for this ECS Service.
TaskDefinition points to the TaskDefinition which contains container related information. Following is the Cloudformation template for the TaskDefinition.

a) In the AWS TaskDefinition part of the template note that
ContainerDefinitions -> Image specifies the image URL of the ECR. Note that the value for this parameter is passed by the AWS CodePipeline’s Deploy stage as discussed in previous section.




****Deployment*****



1. Go to the Github repo link and fork it to your Github account.
You need to provide access from Github to AWS. This can be done using an Access token. Please refer to this link and generate a token for your Github account.
2. Download the file CodePipeline.yml file under the Cloudformation directory and open it.
3. Replace USERNAME, BRANCH andGITHUB ACCESS TOKEN values with your values.
4. Go to Cloudformation services in AWS and create a new stack.
5. Upload the CodePipeline.yml file you just edited as displayed below and click Next.
6.Please make sure the region you are deploying in has at least two availability zones as the 2 subnets inside the VPC (defined in Fargate-Cluster.yml) will be deployed in “a” and “b” availability zones respectively. You can find a list of availability zones from this link.
7. Enter a stack name and click Next. I have specified “CodePipelineStack”.
8. In Review section, click the following checkbox as displayed below.
This is due to the fact that CodePipeline.yml Cloudformation template will be creating IAM roles on our behalf and we need to give our explicit permission for that.
9. Finally hit Create Stack and go to Cloudformation service page and you should be able to see your CodePipeline stack being created as displayed below.

After the CodePipeline stack is created, Head over to AWS CodePipeline service and it should have already started the build stage and start pulling container as displayed below.
Once the CodePipeline Build stage is finished, it will start the Deploy stage and start the deployment of the Fargate-Cluster.yml file as displayed below.
After the CodePipeline has finished, if you go to Cloudformation stacks, you’ll see the Fargate-Cluster.yml file stack will be created as a new stack as below. Which means the Fargate ECS Cluster infrastructure was successfully deployed.




**** Testing The ECS Cluster Service ****

After the CodePipeline has finished, Fargate ECS Cluster should be up and running now. We can access the Container service using the Load balancer DNS address.
Go to EC2 Service -> Load Balancers and find the DNS address as displayed below.
Image for post
Open the DNS address in the browser and you’ll the Hello World message in you browser.
