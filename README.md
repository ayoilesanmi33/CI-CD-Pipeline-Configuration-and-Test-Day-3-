# CI-CD-Pipeline-Configuration-and-Test-Day-3-
Here's the updated section organized for a GitHub repository:

---

## Part 1: CI/CD Pipeline Configuration

### Create a Free Account on GitHub and Create a New Repository

1. Create a free account on GitHub if you don't have one.
2. Create a new repository on GitHub called `cloudmart`.

```sh
cd challenge-day2/frontend
<Run GitHub steps>
```

### Push Changes to GitHub

Start by pushing the changes in the CloudMart application source code to GitHub:

```sh
git status
git add -A
git commit -m "app sent to repo"
git push
```

### Configure AWS CodePipeline

1. **Create a New Pipeline:**
    - Access AWS CodePipeline.
    - Start the 'Create pipeline' process.
    - Name: `cloudmart-cicd-pipeline`.
    - Use the GitHub repository `cloudmart-application` as the source.
    - Add the 'cloudmartBuild' project as the build stage.
    - Add the 'cloudmartDeploy' project as the deployment stage.

### Configure AWS CodeBuild to Build the Docker Image

1. **Create a Build Project:**
    - Give the project a name (for example, `cloudmartBuild`).
    - Connect it to your existing GitHub repository (`cloudmart-application`).
    - **Image:** `amazonlinux2-x86_64-standard:4.0`.
    - Configure the environment to support Docker builds. Enable "Enable this flag if you want to build Docker images or want your builds to get elevated privileges".
    - Add the environment variable `ECR_REPO` with the ECR repository URI.
    - For the build specification, use the following `buildspec.yml`:

    ```yaml
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
          - aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws/l4c0j8h9
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

2. **Add the AmazonElasticContainerRegistryPublicFullAccess Permission to ECR in the Service Role:**
    - Access the IAM console > Roles.
    - Look for the role created "cloudmartBuild" for CodeBuild.
    - Add the permission `AmazonElasticContainerRegistryPublicFullAccess`.

### Configure AWS CodeBuild for Application Deployment

1. **Create a Deployment Project:**
    - Repeat the process of creating projects in CodeBuild.
    - Give this project a different name (for example, `cloudmartDeployToProduction`).
    - Configure the environment variables `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` for the credentials of the user `eks-user` in Cloud Build, so it can authenticate to the Kubernetes cluster.

    *Note: In a real-world production environment, it is recommended to use an IAM role for this purpose. In this practical exercise, we are directly using the credentials of the `eks-user` to facilitate the process, since our focus is on CI/CD and not on user authentication at this moment. The configuration of this process in EKS is more extensive. Refer to the Reference section and check "Enabling IAM principal access to your cluster".*

    - For the deployment specification, use the following `buildspec.yml`:

    ```yaml
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

2. **Replace the Image URI on Line 18 of the `cloudmart-frontend.yaml` Files with `CONTAINER_IMAGE`:**
    - Commit and push the changes.

    ```sh
    git add -A
    git commit -m "replaced image uri with CONTAINER_IMAGE"
    git push
    ```

## Part 2: Test Your CI/CD Pipeline

1. **Make a Change on GitHub:**
    - Update the application code in the `cloudmart-application` repository.
    - File `src/components/MainPage/index.jsx` line 93.
    - Commit and push the changes.

    ```sh
    git add -A
    git commit -m "changed to Featured Products on CloudMart"
    git push
    ```

2. **Observe the Pipeline Execution:**
    - Watch how CodePipeline automatically triggers the build.
    - After the build, the deployment phase should begin.

3. **Verify the Deployment:**
    - Check Kubernetes using `kubectl` commands to confirm the application update.

---

Let me know if there's anything else you need!
