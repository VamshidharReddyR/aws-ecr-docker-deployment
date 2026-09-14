# AWS ECR Docker Deployment

A production-style CI/CD pipeline that builds a Dockerized Python application, runs automated tests, pushes the Docker image to **Amazon ECR**, and deploys the image on an AWS EC2 instance using **Jenkins**.

## Architecture

```text
Developer
   |
   v
GitHub
   |
   v
Jenkins
   |
   +--> Install Dependencies
   |
   +--> Run Pytest
   |
   +--> Build Docker Image
   |
   +--> Authenticate to Amazon ECR
   |
   +--> Tag Docker Image
   |
   +--> Push Image to ECR
   |
   +--> Pull and Run ECR Image
   |
   +--> Health Check
   |
   v
AWS EC2
   |
   v
Docker Container
```

## Technologies Used

* AWS EC2
* Amazon ECR
* Jenkins
* Docker
* Python
* Flask
* Pytest
* GitHub
* AWS CLI
* Linux

## Project Structure

```text
aws-ecr-docker-deployment/
├── app/
│   ├── __init__.py
│   └── app.py
├── tests/
│   └── test_app.py
├── .dockerignore
├── .gitignore
├── Dockerfile
├── Jenkinsfile
└── requirements.txt
```

## Application

The project contains a simple Flask application with two endpoints.

### Application endpoint

```text
GET /
```

Response:

```text
AWS ECR Docker Deployment is working!
```

### Health endpoint

```text
GET /health
```

Response:

```json
{
  "status": "healthy"
}
```

## CI/CD Pipeline

The Jenkins pipeline performs the following steps:

### 1. Install Dependencies

Installs the Python dependencies defined in `requirements.txt`.

### 2. Run Tests

Runs the automated Pytest test suite and generates a JUnit test report.

```bash
python3 -m pytest --junitxml=test-results.xml
```

### 3. Build Docker Image

Builds the application Docker image.

```bash
docker build -t aws-ecr-docker-deployment:<BUILD_NUMBER> .
```

### 4. Authenticate to Amazon ECR

Jenkins authenticates Docker with Amazon ECR using the AWS CLI.

```bash
aws ecr get-login-password --region us-east-1
```

No static AWS access keys are stored in the Jenkins pipeline.

The EC2 instance uses an IAM role with permissions required to interact with ECR.

### 5. Tag Docker Image

The locally built image is tagged with the ECR repository URI.

```text
<account>.dkr.ecr.us-east-1.amazonaws.com/aws-ecr-docker-deployment:<BUILD_NUMBER>
```

### 6. Push Image to ECR

The tagged image is pushed to the private Amazon ECR repository.

```bash
docker push <ECR_URI>:<BUILD_NUMBER>
```

### 7. Deploy Container

The pipeline stops and removes the previous application container and starts a new container using the image stored in ECR.

### 8. Health Check

The pipeline verifies that the deployed application is responding successfully.

```bash
curl --fail http://localhost:5000/health
```

A successful health check allows the Jenkins build to complete successfully.

## Docker Image Tagging

Docker images are tagged using the Jenkins build number.

For example:

```text
Build #1 → :1
Build #2 → :2
Build #3 → :3
```

This provides unique image versions for each Jenkins build and works well with an immutable ECR repository.

## AWS IAM

The EC2 instance uses an IAM role for AWS authentication instead of storing long-term AWS access keys on the server.

The role provides the permissions required for:

* Amazon ECR authentication
* Pulling Docker images from ECR
* Pushing Docker images to ECR

The ECR repository itself is created separately, while the EC2/Jenkins workload uses the role for image operations.

## Local Setup

### Clone the repository

```bash
git clone https://github.com/VamshidharReddyR/aws-ecr-docker-deployment.git
cd aws-ecr-docker-deployment
```

### Create a Python virtual environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### Install dependencies

```bash
pip install -r requirements.txt
```

### Run tests

```bash
pytest
```

### Build Docker image

```bash
docker build -t aws-ecr-docker-deployment:local .
```

### Run the container

```bash
docker run -d \
  --name aws-ecr-docker-deployment \
  -p 5000:5000 \
  aws-ecr-docker-deployment:local
```

### Test the application

```bash
curl http://localhost:5000/
```

Health check:

```bash
curl http://localhost:5000/health
```

## Jenkins Configuration

Jenkins is configured to use:

```text
Pipeline script from SCM
```

Repository:

```text
https://github.com/VamshidharReddyR/aws-ecr-docker-deployment.git
```

Branch:

```text
main
```

Pipeline definition:

```text
Jenkinsfile
```

## Key DevOps Practices Demonstrated

* CI/CD automation with Jenkins
* Git-based source control
* Automated application testing
* Docker image creation
* Amazon ECR integration
* AWS IAM role-based authentication
* Immutable Docker image versioning
* Container deployment on EC2
* Automated post-deployment health checks
* JUnit test reporting

## Result

Every successful Jenkins build produces a versioned Docker image in Amazon ECR and deploys that image to the EC2 environment after passing automated tests.

```text
GitHub → Jenkins → Test → Docker Build → ECR Push → Deploy → Health Check
```
