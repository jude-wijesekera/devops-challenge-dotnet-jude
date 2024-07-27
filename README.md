# CI/CD Pipeline for .NET 5 Application with Trivy and SonarCloud Integration

## Overview

This documentation outlines the steps to set up a CI/CD pipeline for a .NET 5 application using GitHub Actions. The pipeline includes building and pushing a Docker image, scanning for vulnerabilities using Trivy, performing code quality and security analysis using SonarCloud, and running integration tests on the built Docker image.

## Prerequisites

- **GitHub Repository**: Ensure you have a GitHub repository set up with your .NET 5 application.
- **DockerHub Account**: Create a DockerHub account and generate an access token.
- **SonarCloud Account**: Create a SonarCloud account, set up a project, and generate an access token.
- **Trivy**: Trivy is an open-source vulnerability scanner for containers and other artifacts.
- **Secrets**: Add the following secrets to your GitHub repository:
  - `DOCKERHUB_USERNAME`: Your DockerHub username.
  - `DOCKERHUB_TOKEN`: Your DockerHub access token.
  - `SONAR_TOKEN`: Your SonarCloud access token.
  - `SONAR_PROJECT_KEY`: Your SonarCloud project key.
  - `SONAR_ORGANIZATION`: Your SonarCloud organization.

## Directory Structure

Ensure your repository follows a directory structure similar to the following:

```plaintext
.
├── src
│ ├── DevOpsChallenge.SalesApi
│ │ ├── Dockerfile
│ │ ├── DevOpsChallenge.SalesApi.csproj
│ │ └── ... (other source files)
│ └── ... (other projects)
├── tests
│ ├── DevOpsChallenge.SalesApi.Business.UnitTests
│ │ ├── DevOpsChallenge.SalesApi.Business.UnitTests.csproj
│ │ └── ... (test files)
│ ├── DevOpsChallenge.SalesApi.IntegrationTests
│ │ ├── DevOpsChallenge.SalesApi.IntegrationTests.csproj
│ │ └── ... (test files)
└── .github
└── workflows
└── ci.yml
```

## Docker File creation

Create a Dockerfile (`./src/Dockerfile`) with the following content:

```dockerfile
# Use the official .NET 5 SDK image as the build environment
FROM mcr.microsoft.com/dotnet/sdk:5.0 AS build
WORKDIR /src

# Copy the .csproj files and restore any dependencies
COPY DevOpsChallenge.SalesApi.Business/*.csproj ./DevOpsChallenge.SalesApi.Business/
COPY DevOpsChallenge.SalesApi.Database/*.csproj ./DevOpsChallenge.SalesApi.Database/
COPY DevOpsChallenge.SalesApi/*.csproj ./DevOpsChallenge.SalesApi/
RUN dotnet restore ./DevOpsChallenge.SalesApi/DevOpsChallenge.SalesApi.csproj

# Copy the rest of the project files
COPY DevOpsChallenge.SalesApi.Business ./DevOpsChallenge.SalesApi.Business/
COPY DevOpsChallenge.SalesApi.Database ./DevOpsChallenge.SalesApi.Database/
COPY DevOpsChallenge.SalesApi ./DevOpsChallenge.SalesApi/

# Build the application
WORKDIR /src/DevOpsChallenge.SalesApi
RUN dotnet publish -c Release -o /app/publish

# Use the official ASP.NET Core runtime image as the runtime environment
FROM mcr.microsoft.com/dotnet/aspnet:5.0 AS runtime
WORKDIR /app

# Copy the build output from the build environment
COPY --from=build /app/publish .

# Expose the port the application runs on
EXPOSE 80
EXPOSE 90

# Set the entry point to run the application
ENTRYPOINT ["dotnet", "DevOpsChallenge.SalesApi.dll"]
```

## GitHub Actions Workflow

Create a GitHub Actions workflow file (`.github/workflows/ci.yml`) with the following content:

## YAML Configuration

Below is the configuration used for GitHub Actions CI/CD pipelines:

```yaml
name: CI-build-and-push-devops-challenge

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v6
        with:
          context: ./src
          file: ./src/Dockerfile
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/dockerfordotnetapi:latest, ${{ secrets.DOCKERHUB_USERNAME }}/dockerfordotnetapi:${{ github.run_number }}

  scan-vulnerabilities:
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Trivy
        run: |
          sudo apt-get update
          sudo apt-get install -y wget apt-transport-https gnupg lsb-release
          wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
          echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
          sudo apt-get update
          sudo apt-get install -y trivy

      - name: Scan code for vulnerabilities
        id: trivy-code-scan
        run: |
          trivy fs --severity HIGH,CRITICAL --format table --output trivy-code-report.txt .
          cat trivy-code-report.txt

      - name: Scan Docker image for vulnerabilities
        id: trivy-image-scan
        run: |
          trivy image ${{ secrets.DOCKERHUB_USERNAME }}/dockerfordotnetapi:latest --severity HIGH,CRITICAL --format table --output trivy-image-report.txt
          cat trivy-image-report.txt

      - name: Upload Trivy scan results
        uses: actions/upload-artifact@v2
        with:
          name: trivy-scan-results
          path: |
            trivy-code-report.txt
            trivy-image-report.txt

  sonarcloud:
    runs-on: ubuntu-latest
    needs: scan-vulnerabilities

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v1
        with:
          java-version: '11'

      - name: Setup .NET
        uses: actions/setup-dotnet@v1
        with:
          dotnet-version: '5.0.x'

      - name: Cache SonarCloud packages
        uses: actions/cache@v2
        with:
          path: ~/.sonar/cache
          key: ${{ runner.os }}-sonar

      - name: Install SonarScanner for .NET
        run: dotnet tool install --global dotnet-sonarscanner

      - name: SonarCloud Scan
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          dotnet-sonarscanner begin /k:"jude-wijesekera_devops-challenge-dotnet-jude" /o:"jude-wijesekera" /d:sonar.login="${{ secrets.SONAR_TOKEN }}"
          dotnet build
          dotnet-sonarscanner end /d:sonar.login="${{ secrets.SONAR_TOKEN }}"

  start-and-test:
    runs-on: ubuntu-latest
    needs: sonarcloud

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Pull Docker image if not available locally
        run: |
          docker pull ${{ secrets.DOCKERHUB_USERNAME }}/dockerfordotnetapi:latest

      - name: Start Docker container
        run: |
          docker run -d --name devops-challenge-container -p 8080:80 ${{ secrets.DOCKERHUB_USERNAME }}/dockerfordotnetapi:latest

      - name: List all containers (for debugging)
        run: |
          docker ps -a

      - name: Wait for container to be ready
        run: |
          echo "Waiting for the container to be ready..."
          sleep 15

      - name: Set up .NET
        uses: actions/setup-dotnet@v1
        with:
          dotnet-version: '5.0.x'

      - name: Restore dependencies
        run: dotnet restore src/DevOpsChallenge.SalesApi/DevOpsChallenge.SalesApi.csproj

      - name: Build
        run: dotnet build src/DevOpsChallenge.SalesApi/DevOpsChallenge.SalesApi.csproj --no-restore

      - name: Business Test
        run: dotnet test tests/DevOpsChallenge.SalesApi.Business.UnitTests/DevOpsChallenge.SalesApi.Business.UnitTests.csproj --no-build --verbosity normal

      - name: Integration Test
        run: dotnet test tests/DevOpsChallenge.SalesApi.IntegrationTests/DevOpsChallenge.SalesApi.IntegrationTests.csproj --no-build --verbosity normal

      - name: Stop Docker container
        run: |
          CONTAINER_ID=$(docker ps -q -f name=devops-challenge-container)
          if [ -n "$CONTAINER_ID" ]; then
            docker stop $CONTAINER_ID
          else
            echo "Container devops-challenge-container is not running."
          fi

```

## Workflow Steps Explanation :blue_book:

### Build Job:

- **Checkout :** Checks out the repository.
- **Set up QEMU :** Sets up QEMU for cross-platform builds.
- **Set up Docker Buildx :** Sets up Docker Buildx for building multi-platform Docker images.
- **Login to DockerHub :** Logs into DockerHub using the provided secrets.
- **Build and push :** Builds and pushes the Docker image to DockerHub.

### Scan Vulnerabilities Job:

- **Checkout code :** Checks out the repository.
- **Set up Trivy :** Installs Trivy for vulnerability scanning.
- **Scan code for vulnerabilities :** Scans the codebase for vulnerabilities and outputs the results.
- **Scan Docker image for vulnerabilities :** Scans the built Docker image for vulnerabilities and outputs the results.
- **Upload Trivy scan results :** Uploads the Trivy scan results as artifacts.

### SonarCloud Job:

- **Checkout code :** Checks out the repository.
- **Setup Java :** Sets up Java, which is required by SonarCloud.
- **Setup .NET :** Sets up .NET SDK.
- **Cache SonarCloud packages :** Caches SonarCloud packages for faster builds.
- **Install SonarScanner for .NET :** Installs the SonarScanner tool for .NET.
- **SonarCloud Scan :** Runs the SonarCloud analysis on the codebase.

### Start and Test Job:

- **Checkout code :** Checks out the repository.
- **Pull Docker image :** Pulls the Docker image if it's not available locally.
- **Start Docker container :** Starts the Docker container.
- **List all containers :** Lists all running containers for debugging purposes.
- **Wait for container to be ready :** Waits for the container to be ready.
- **Set up .NET :** Sets up the .NET SDK.
- **Restore dependencies :** Restores the .NET project dependencies.
- **Build :** Builds the .NET project.
- **Business Test :** Runs the business unit tests.
- **Integration Test :** Runs the integration tests.
- **Stop Docker container :** Stops the Docker container.

## Best Practices

**Modular Jobs :** Each job in the workflow focuses on a specific task, making the workflow modular and easier to maintain.

**Secrets Management :** Sensitive information like DockerHub credentials and SonarCloud tokens are stored securely using GitHub Secrets.

**Caching :** Caching is used to speed up the workflow by avoiding repetitive installations.

**Dependency Management :** Dependencies are restored and cached to ensure consistency across different runs.

**Vulnerability Scanning :** Trivy is used to scan both the codebase and the Docker image for vulnerabilities.

**Code Quality Analysis :** SonarCloud is used for code quality and security analysis, ensuring adherence to best practices.

## Conclusion

This documentation provides a comprehensive guide to setting up a CI/CD pipeline for a .NET 5 application using GitHub Actions. The pipeline includes building and pushing a Docker image, scanning for vulnerabilities using Trivy, performing code quality analysis using SonarCloud, and running integration tests. By following this guide, you can ensure a high-quality deliverable and a streamlined developer experience.
