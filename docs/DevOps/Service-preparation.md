# Preparing your microservice for deployment

This document explains the steps needed in order to prepare your service to be deployed to the production environment. Before going through this document please make sure you have covered the [Cluster Setup](https://github.com/Geeksltd/Olive/blob/master/docs/Microservices/DevOps/Cluster-setup.md), [Docker](https://github.com/Geeksltd/Olive/blob/master/docs/Microservices/DevOps/Docker.md), [Jenkins](https://github.com/Geeksltd/Olive/blob/master/docs/Microservices/DevOps/Jenkins.md) and [Security](https://github.com/Geeksltd/Olive/blob/master/docs/Microservices/DevOps/Security.md) documents to get a better undestanding of the production environment and the CI/CD processes.

In order to deploy a service we need to be able to build the code and create the docker image and the resources we need to be able to update the production cluster, so let's start off with setting up the CI/CD in Jenkins.

## CI/CD
For more information about how the CI/CD project works please refere to [this](https://github.com/Geeksltd/Olive/blob/master/docs/Microservices/DevOps/Jenkins.md) document. 
We need the following files to be able to setup Jenkins. 
* Jenkinsfile (used by Jenkins)
* Deployment.yaml (for Kubernetes)
* Sercret.yaml (for Kubernetes)
* Service.yaml (for Kubernetes)

To automate, and possibly avoid human error, creating those files a script has been provided in the BigPicture repository in /DevOps called ContinuousDelivery.bat. In the same folder there are two other folders called Jenkins and Kubernetes which contains the templates used by ContinuousDelivery.bat script to generate the DevOps files. When running the script you need to provide your service name. This can be achived by passing the name with -serviceName argument when calling the script. Bear in mind that the name you provide here will be used in the CI/CD process and all the resources such as Service, Deployment, Labels on Kubernetes. 
When you run the script it creates a directory called Generated in the same directory as the script file. You need to copy the Generated/YourServiceName folder to the root of the Jenkins repository and rename it to DevOps so that the Jenkins job we create later on in this document can have access to it. Commit the changes and push it to your remote repository.

### Jenkins
To complete the CI/CD setup we need to create a Jenkins job on the Jenkins server. 

* Login to Jenkins
* Navigate to New Item on dashboard
* Provide a name for for your job
* Select Pipeline (or if you already have a service installed on Jenkins you can use the "copy from" option)
* Click next
* In the Pipeline section select "Pipeline script from SCM" 
* Select Git as SCM
* Set the Repository Url to the Jenkins repository URL
  * You need to create a user in your repository for Jenkins which has Read permission to the Jenkins repository. Add the credentials by clicking the "Add" button. If you already have added it you can select it from the dropdown.
* Leave the Branch Specifier as */master
* Leave Repository browser as "Auto"
* Set the Script Path to "YourService/Jenkinsfile
* Click Save

### Jenkinsfile
We need to create a Jenkinsfile for the service. The template can be found in [Jenkins](https://github.com/Geeksltd/Olive/blob/master/docs/Microservices/DevOps/Jenkins.md).
/// TODO: Jenkinsfile to be generated by the ContinuousDelivery.bat file.
Create a new copy and edit all the placeholders with $PLACEHOLDER_NAME$ pattern. Currently there are two placeholders listed below but they may change :
* $ROOT_ECR_URL$
* $GIT_REPOSITORY_URL$

After making sure the Jenkinsfile is ready, copy it to the Jenkins repository under /{YourService}/ directory. The path and filename are important as the Jenkins master will look for /{YourService}/Jenkinsfile (as configured in the previous section).

### Build Credentials
If you look at the Jenkinsfile there is a credentials id called {YOUR_SERVICE}\_GIT\_CREDENTIALS\_ID. Jenkins will use that ID to load the service git repository credentials to pull the source code. You need to create a user in your repository for Jenkins which has Read permission to your live branch. Once created, add the credentials to Jenkins:

* Navigating to the live Jenkins
* From the left menu, select Credentials and then System
* If you already have created a domain for your service, navigate to the domain. If not create one by clicking "Add domain" (in the left menu)
* Add a new username/password credentials record and put the git credentials there. Set {YOUR_SERVICE}\_GIT\_CREDENTIALS\_ID as the ID.


## Security 
// TODO: We can create a script to do the followings.

### Secrets
As mentioned in the [Security](https://github.com/Geeksltd/Olive/blob/master/docs/Microservices/DevOps/Security.md) document, the production secure info should be stored in AWS Secrets Manager an be pulled during startup on the production. 
Instruction :
* Login to the AWS console. 
* Make sure you are in the correct region
* Navigate to Secrets Manager
* Click "Create a new secret"
* Select "Other type of secrets" as secret type
* Select Plaintext and create the same json structure for your secure info as you would in the appsettings.Production.json
* Leave Secret the encryption key as default.
* Provide a name and description. 
* Click Save

Examle of a secret template :
```json
{
  "Microservice": {
    "Hub": {
      "AccessKey": "%HUB_SERVICE_ACCESS_KEY%"
    }
  },
  "ConnectionStrings": {
    "Default": "%CONNECTION_STRING%"
  }
}

```

### IAM Role (Service Runtime Identity)
To identify our service in our infrastructure we need to create an IAM Role for it as suggested [here](https://github.com/Geeksltd/Olive/blob/master/docs/Microservices/DevOps/Security.md). 
Instruction :
* Login to AWS
* Navigate to IAM > Roles
* Click "new role"
* Select AWS service as the service type
* Select EC2 as the service that will use this role
* Click next
* Click next, skip assigning policies for now.
* Provide a name : {YourService}Runtime and description if needed.
* Click Create role

### IAM Role permissions
Now that we have an IAM role for the service we need to assign it. For each permission we need to have a policy. There are some mandatory policies we have to have and if the service uses any of AWS resources such as S3 we have to create separate policies.

The mandatory policies are :

### KMS Data Key Generation\Decryption 
In each application there is one service which provides "log in" to users in the system and other services will only need to authenticate users. The service that logs the users into the system needs to be able to generate KMS Data Key. If the service you are deploying is the login provider you need to create a policy which grants KMS Data Key Generation permission to the service and for that you can use the following template:

```json

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "kms:Decrypt",
                "kms:GenerateDataKey"
            ],
            "Resource": "%KMS_ARN%"
        }
    ]
}

```

If your service only needs to decrypt authentication tokens you can use :

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "kms:Decrypt",
            "Resource": "%KMS_ARN%"
        }
    ]
}
```

Before createing a KMS policy, specially the one for decryption, make sure it doesn't exist to avoid duplication.

Once you have a template, you should:
* Login to the AWS console
* Navigate to IAM > Policies
* Click "Create policy"
* Click on the Json tab
* Paste your template
* Click next
* Provide a meaningful name based on wether it is for decryption or key generation
* Click save

### Secrets Manager Secure Info
We also need to create a policy to grant permission to the service IAM role to be able to read the application secure info. The creation process is the same as above and the template is :

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "secretsmanager:GetSecretValue",
            "Resource": "%SERVICE_SECURE_INFO_SERCRET_ARN%"
        }
    ]
}

```


If you need any more permissions you may follow the same process.

### Assigning
Now that we have all the policies we need we can assign them to the service IAM role.

* Go to IAM in AWS console
* Navigate to Roles
* Search for your service role and click on it
* Click "Attach policies"
* Select all the policies you have created. Make sure you have selected the correct filter type on the top left hand side.
* Click Attach policy

## Kubernetes Metadata
// TODO : We should use Helm to create a package for each service to be able to organize its dependencies and linked resources.
By now everything should be set up and we should be ready for the final step, which is introducing the service to Kubernetes.
To do that we need to :
* Add a secret for the service's runtime AWS IAM role.
  * Go to LocalPathToService\DevOps\Kubernetes
  * Update Secret.yaml and replace %ROLE_ARN% and %REGION% with the values of the IAM role we created earlier in this document. __IMPORTANT: Do not, I repeat, do not, and I repeat again, do not commit this change. Make sure you discard these changes after applying them to Kubernetes__
  * Whilst you are in the DevOps folder, run "kubectl apply -f Secret.yaml". It should create the secret in Kubernetes. To check that, run "kubectl get secrets" and the secret should show up.
* Add a service record for the application service
  * Double check the Service.yaml file and make sure everthing is OK.
  * Run "kubectl run apply -f Service.yaml"
  * Double check the service has been created by running "kubectl get services" and check if the service exists.
* Accessing the service
  * We need to set up our Ingress configuraiton to be able to redirect the internet traffic to our service.
  * Navigate to your local BigPicture repository > Cluster and edit 3_IngressRules.yaml
  * Add a new rule for your service. Following the same naming convention as the other rules. You can find the rule template below:
  ```yaml
  - host: SERVICE_URL
    http:
      paths:
      - backend:
          serviceName: SERVCIE_NAME
          servicePort: 80 
  ```
  This template tells our Ingress to redirect all the http request to SERVICE_URL:80 to the service we created in the previous step.
  * Commit and push the changes.
  * Run "kubectl apply -f path_to_your_local_BigPicture/Cluster/3_IngressRules.yaml"
  
You are all set. Now you should be able to run a build on Jenkins and once done you should be able to see your service on live. 
  
  
  
  
  
