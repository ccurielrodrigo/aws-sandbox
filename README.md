Here’s the requested README content and comments for the GitHub Actions workflow:

---

### README (raw code)

```markdown
# EC2 Container Deployment Showcase

This project demonstrates how to deploy a containerized application to an EC2 instance. It includes a GitHub Actions workflow that automates the deployment process when changes are pushed to a specific branch.

## Purpose

The purpose of this project is to illustrate the steps needed to deploy a Docker container to an EC2 instance using GitHub Actions, enabling a streamlined and automated deployment pipeline.

## Getting Started

To deploy the container, follow these steps:

1. **Pull the Container:**
   - Clone this repository and navigate to the project directory.
   - Ensure Docker is installed and running on your local machine.
   
   ```bash
   git clone <repository-url>
   cd <repository-folder>
   ```

2. **Review the Deployment Workflow:**
   - Open `.github/workflows/deploy.yml` to understand the GitHub Actions workflow.
   - The workflow is triggered by changes pushed to the `qf-26-auto-aws-deploy` branch and handles deployment to the specified EC2 instance.

3. **Make and Push Changes:**
   - If you want to deploy updates, make changes to the codebase and push them to the `qf-26-auto-aws-deploy` branch. This will automatically trigger the workflow to redeploy the container on EC2.
   
   ```bash
   git add .
   git commit -m "Your commit message"
   git push origin qf-26-auto-aws-deploy
   ```

4. **Access the Deployed Application:**
   - Visit the AWS Console to check deployment status: [AWS Console (ap-northeast-1)](https://ap-northeast-1.console.aws.amazon.com/console/home?region=ap-northeast-1).
   - Open the public IP address of the `stg-test` EC2 instance to view the deployed application: [http://ec2-52-194-219-13.ap-northeast-1.compute.amazonaws.com/](http://ec2-52-194-219-13.ap-northeast-1.compute.amazonaws.com/).

## Requirements

- **Docker**: Ensure Docker is installed to run and test the container locally.
- **GitHub Access**: Have permissions to push changes to the `qf-26-auto-aws-deploy` branch.
- **AWS EC2 Access**: Access to the AWS Console to monitor deployment and retrieve the public IP address of the `stg-test` instance if necessary.

## Workflow Overview

The GitHub Actions workflow (`deploy.yml`) is designed to:

1. Build the Docker container.
2. Deploy the container to the specified EC2 instance.
3. Notify upon successful deployment.

Refer to `.github/workflows/deploy.yml` for more details on the automation steps.

---

### Notes

- Ensure that any sensitive information, such as AWS credentials, is securely stored and referenced in the workflow via GitHub Secrets.
```

---

### Workflow Comments

```yaml
name: Docker CI/CD Pipeline

on:
  push:
    branches:
      - qf-26-auto-aws-deploy  # Triggers on pushes to this branch

jobs:
  build-test-deploy:
    runs-on: ubuntu-latest  # Changed to use GitHub-hosted runner

    steps:
      - name: Checkout code
        uses: actions/checkout@v2  # Action to pull the repository's code
        with:
          verbose: true  # Provides detailed logs for the checkout process

      - name: Log Git Commit Information
        run: |
          echo "Commit SHA: ${{ github.sha }}"  # Logs the SHA of the current commit
          echo "Commit Message: ${{ github.event.head_commit.message }}"  # Logs commit message
          echo "Branch: ${{ github.ref_name }}"  # Logs branch name for verification

      - name: Login to DockerHub
        uses: docker/login-action@v1  # Action to authenticate to DockerHub
        with: 
          username: ${{ secrets.DOCKER_USERNAME }}  # DockerHub username from GitHub Secrets
          password: ${{ secrets.DOCKER_PASSWORD }}  # DockerHub password from GitHub Secrets

      - name: Verify Docker Login
        run: docker info  # Checks Docker login success and details
        continue-on-error: false  # Fails the workflow if Docker login fails

      - name: Display DockerHub Environment Variables
        run: |
          echo "DockerHub Username: ${{ secrets.DOCKER_USERNAME }}"  # Shows DockerHub username
          echo "DockerHub Token (obscured): ********"  # Masks DockerHub token for security

      - name: Build Docker Image
        uses: docker/build-push-action@v2  # Action to build and push Docker image
        with:
          context: .  # Builds from current directory context
          push: true  # Pushes the built image to DockerHub
          tags: ccurielrodrigo/webapp:${{ github.sha }}  # Tags image with SHA for tracking
        env:
          BUILDKIT_PROGRESS: plain  # Sets verbose build output for Docker

      - name: Verify Docker Image Build and Push
        run: docker images  # Lists images to confirm successful build and push
        continue-on-error: false  # Fails the workflow if image is not found

      - name: Deploy to EC2 Instance
        uses: appleboy/ssh-action@master  # Action to SSH into EC2 and run deployment script
        with:
          host: ${{ secrets.EC2_HOST }}  # EC2 host from GitHub Secrets
          username: ${{ secrets.EC2_USERNAME }}  # EC2 SSH username
          key: ${{ secrets.EC2_PRIVATE_KEY }}  # Private key for SSH authentication
          script: |
            echo "Checking and installing Docker if not found"
            if ! command -v docker &> /dev/null; then  # Installs Docker if missing
              echo "Docker not found. Installing Docker..."
              sudo yum update -y
              sudo yum install -y docker
              sudo service docker start
              sudo systemctl enable docker
              sudo usermod -aG docker $USER
              newgrp docker
            fi
            echo "Stopping existing Docker container (if any)"
            sudo docker stop webapp || true  # Stops any running container named 'webapp'
            echo "Removing existing Docker container (if any)"
            sudo docker rm webapp || true  # Removes any existing container named 'webapp'
            echo "Pulling new Docker image from DockerHub"
            sudo docker pull ${{ secrets.DOCKER_USERNAME }}/webapp:${{ github.sha }}  # Pulls latest image
            echo "Running new Docker container with the latest image"
            sudo docker run -d --name webapp -p 80:80 ${{ secrets.DOCKER_USERNAME }}/webapp:${{ github.sha }}  # Deploys new container
          continue-on-error: false  # Fails workflow if deployment script errors
        env:
          SSH_KNOWN_HOSTS: ${{ secrets.EC2_HOST }}  # Adds EC2 host to known hosts for secure SSH
```

These comments explain each step, action, and option used within the GitHub Actions workflow to clarify what each part accomplishes in the CI/CD pipeline.