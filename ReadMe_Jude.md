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

