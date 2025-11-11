#### This is a basic FastAPI backend app with database and CRUD operations. Terraform is used to set up AWS infrastructure. CI/CD pipeline configured via Github actions.

# Project Setup
### CI/CD overview
* Infrastructure provisioning via Terraform has been integrated as part of CI/CD pipeline.
On every push to main, CI/CD workflow sets up infrastructure _(EC2 instance, MySQL DB, etc)_ if it is not already set up.
* Build and push docker image to Docker Hub
* SSH into EC2 instance and pull the latest image during deployment


### Set secret keys on Github
CI/CD pipeline will require following secret keys to work:
```
- AWS_ACCESS_KEY
- AWS_SECRET_ACCESS_KEY
- DOCKERHUB_PASSWORD
- DOCKERHUB_USERNAME
```

`AWS_ACCESS_KEY` and `AWS_SECRET_ACCESS_KEY` will be required by Terraform to set up AWS resources.  
`DOCKERHUB_PASSWORD` and `DOCKERHUB_USERNAME` will be used to sign in to Docker Hub and push latest image.


Once secret keys are set up, push to the main branch, or manually trigger deployment workflow to ensure everything is working great!


# Overview
### Terraform setup
* We will be using Terraform’s remote backend feature to save the state file in S3 bucket. This is important because we need to integrate terraform as part of the CI/CD pipeline. Since the state file will be in a shared, centralized location, the CI/CD workflow will be able to fetch the state file and tally the current state of the infrastructure.
* For Terraform’s remote backend, we need to create an S3 bucket _(this is where the state file will be stored)_ and a DynamoDB table _(for locking to prevent concurrent modifications)_. These resources can be created manually using AWS console, however I’ve created them using a terraform script that can be found in `/infra/remote-be/main.tf`
* The actual file that sets up infrastructure for the FastAPI app is `infra/main.tf`. It creates following resources:
  * MySQL database (AWS, RDS)
  * EC2 instance
  * A key-pair that can be used to ssh to the EC2 instance
  * Relevant security groups


### CI/CD Workflow
Following steps are executed in the Github actions runner (VM).  
Ensure secrets are properly configured.

* Checkout code
* Download Terraform on the Github VM / runner
* Execute `terraform apply` to set up infrastructure, if not already set up
* Store DB address, EC2 IP and key-pair in github env as these values will be required in later steps
* Build & Push docker image to Docker Hub
* SSH into the EC2 instance
  * Pull the latest image from Docker Hub
  * Run the container


### Running this application on a different AWS account
1. Create an IAM user for the new AWS account.
    * Save its `AWS_ACCESS_KEY` & `AWS_SECRET_ACCESS_KEY`
    * Go to Github secrets and set these credentials as repository secrets
  
2. In your new AWS account, create an S3 bucket and DynamoDB table to store Terraform’s remote state file
    * You can use the file `infra/remote-be/main.tf` to do this. Just give unique values to `bucket_name` & `dynamodb_table_name` variables.
    * In the app’s main infrastructure file, which is `infra/main.tf`, specify the bucket name and dynamodb_table name in the backend block (make sure you replace the `bucket` & `dynamodb_table` with the actual resource names you created in previous step):
         <img width="450" height="240" alt="remote-be" src="https://github.com/user-attachments/assets/751c3cc3-8669-46be-9314-090bbb281073" />

    * Push your changes of `infra/main.tf` to the main branch. The push to the main branch will trigger the CI/CD pipeline. Since new AWS account credentials have already been provided in github secrets, the CI/CD pipeline will set up the infrastructure in this AWS account and will deploy the app there.


