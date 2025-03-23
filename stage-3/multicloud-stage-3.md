# MultiCloud, DevOps & AI Challenge - Stage 3

 In this third stage, I achieve the following:
 - Configure CI/CD using AWS codebuild and AWS codepipeline
 - Utilized git for version control

## Step 1 CI/CD Pipeline Configuration
1. First create a repository on github called `cloudmart`
Follow steps in git to connect the local repo.

2. Push the changes in the Cloudmart application source code to github: (We are creating the pipeline for the frontend)

```sh
git status
git add -A
git commit -m "app sent to repo"
git push
```

3. Configure AWS CodePipeline:
- Go to the AWS CodePipeline console and select **Create a New Pipeline:**
    -  Under "Choose creation option", select **Build custom pipeline** then select **Next**
    ![choose creation option](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/choose%20creation%20option.png)

    - Under "Choose pipeline settings", name the pipeline ` cloudmart-cicd-pipeline` . Leave the default settings. Then, select **Next**

![choose pipeline settings](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/choose%20pipeline%20settings.png)
- In the "Add source stage", Choose **GitHub (via OAUth app)**, for simplicity and Choose **Connect to Github**. Confirm the connection via OAuth and select the `cloudmart` repository` and main branch as the source.

![add source stage 1](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/add%20source%20stage%201.png)
![add source stage 2](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/add%20source%20stage%202.png)
![add source stage 3](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/add%20source%20stage%203.png)
![add source stage 4](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/add%20source%20stage%204.png)

- Under the "Add build stage", select **Other build providers**, then select **AWS CodeBuild**, Choose **Create project** (This will direct you to another window, where you can configure AWS cloudbuild settings). Name the build project, `cloudmartBuild`

![create new build project](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/create%20new%20build%20project.png)

- Add the 'cloudmartBuild' project you created as the build stage.

- Skip the test and the deploy stage for now

- Review the settings and create the pipeline

![Create pipeline](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/Create%20pipeline.png)



4. For the AWS CodeBuild configuration to build the Docker Image, use the following configurations, leaving the rest as default

    - Give the project a name (for example, **`cloudmartBuild`**).
    - Connect it to your existing GitHub repository (**`cloudmart`**).
    - **Image: `amazonlinux2-x86_64-standard:4.0**
    - Configure the environment to support Docker builds. Enable "Enable this flag if you want to build Docker images or want your builds to get elevated privileges"
    - Add the environment variable **ECR_REPO** with the ECR repository URI.
    - For the build specification, use the following **`buildspec.yml`**. (Replace the ecr command in the buildspec with the one from your ECR push commands):

```sh
version: 0.2
phases:
  install:
    runtime-versions:
      docker: 20
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - REPOSITORY_URI=$ECR_REPO
      - aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws/g2z2j1n0
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - export imageTag=$CODEBUILD_RESOLVED_SOURCE_VERSION
      - printf '[{\"name\":\"cloudmart-app\",\"imageUri\":\"%s\"}]' $REPOSITORY_URI:$imageTag > imagedefinitions.json
      - cat imagedefinitions.json
      - ls -l

env:
  exported-variables: ["imageTag"]

artifacts:
  files:
    - imagedefinitions.json
    - cloudmart-frontend.yaml

```
![AWS codebuild setting](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/AWS%20codebuild%20setting.png)
![AWS codebuild setting 2](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/AWS%20codebuild%20setting%202.png)
![AWS codebuild setting 3](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/AWS%20codebuild%20setting%203.png)
![AWS codebuild setting 4](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/AWS%20codebuild%20setting%204.png)


When we first run the build, It fails with an error:
![command error](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/command%20error.png)

The reason for this error is authorization. We must ensure that the cloudmart role that was created has full access to ECR to perform the build actions

Resolve this by adding the `AmazonElasticContainerRegistryPublicFullAccess` permission to ECR in the service role. To do this:

- Access the IAM console > Roles.
- Look for the role created "cloudmartBuild" for CodeBuild.
![iam cloudmart service role](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/iam%20cloudmart%20service%20role.png)

- Select **Add permissions**, then **attach policies**

![add permissions cloudmart service role](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/add%20permissions%20cloudmart%20service%20role.png)

Search for and select 
**AmazonElasticContainerRegistryPublicFullAccess** 
![add Ecr permissions](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/add%20ECR%20permissions.png)

![policy attached to role](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/policy%20attached%20to%20role.png)

When we run the build again, it will be successful
![build success](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/build%20success.png)

![build success 2](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/build%20success%202.png)

5. Configure AWS CodeBuild for Application Deployment

Add a new stage within the pipeline under the build stage named `Deploy`. 
![edit pipeline](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/edit%20pipeline.png)

![add stage](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/add%20stage.png)

Then click on **Add action group**
![deploy stage added. add action group](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/deploy%20stage%20added.png)

![deploy stage added 2](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/deploy%20stage%20added%20(2).png)

![deploy](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/deploy.png)

In the edit action dialogue box, configure:
   - Name: `Deploy`
   - For the action provider, select AWS CodeBuild
   - Input artifact: `Build artifact`
   - Project name: Choose **Create project**

Repeat the process of creating projects in CodeBuild.
Give the project the name `cloudmartDeployToProduction`.

- Configure the environment variables AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY for the credentials of the user eks-user in Cloud Build, so it can authenticate to the Kubernetes cluster.


Note: In this test environment, we are using the credentials of the eksuser for authentication. it is recommended to use an IAM role for this purpose in a real ptroduction environment. 

For the deployment specification, use the following buildspec.yml:

```sh
version: 0.2

phases:
  install:
    runtime-versions:
      docker: 20
    commands:
      - curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd64/kubectl
      - chmod +x ./kubectl
      - mv ./kubectl /usr/local/bin
      - kubectl version --short --client
  post_build:
    commands:
      - aws eks update-kubeconfig --region us-east-1 --name cloudmart
      - kubectl get nodes
      - ls
      - IMAGE_URI=$(jq -r '.[0].imageUri' imagedefinitions.json)
      - echo $IMAGE_URI
      - sed -i "s|CONTAINER_IMAGE|$IMAGE_URI|g" cloudmart-frontend.yaml
      - kubectl apply -f cloudmart-frontend.yaml


```

Finally, Click **Done**, then **Save**.

![deploy stage added ](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/deploy%20stage%20added%20(2).png)

- Replace the image URI on line 18 of the **`cloudmart-frontend.yaml`** files with CONTAINER_IMAGE.

- Commit and push the changes.


## Step 2 Test your CI/CD Pipeline
1. 1. **Make a Change on GitHub:**
    - Update the application code in the **`cloudmart-application`** repository.
    - File `src/components/MainPage/index.jsx` line 93
    - Commit and push the changes.

```sh
git add -A
git commit -m "changed to Featured Products on CloudMart"
git push
```

![deploy is successful](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/deploy%20is%20successful.png)

![featured product changed](https://github.com/laraadeboye/multicloud-devops-ai-challenge/blob/doc/update-readme/stage-3/images/Featured%20product%20changed.png)

2. **Observe the Pipeline Execution:**
    - Watch how CodePipeline automatically triggers the build.
    - After the build, the deployment phase should begin.

3. **Verify the Deployment:**
    - Check Kubernetes using **`kubectl`** commands to confirm the application update.
