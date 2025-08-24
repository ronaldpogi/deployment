# AWS ECS + CodePipeline Deployment Guide

This repository contains the steps I followed to create and deploy an application using **Amazon ECS** with **CodePipeline**.

## Overview
The goal of this setup is to automate the build and deployment process using AWS services such as:
- **ECR** (Elastic Container Registry) for storing Docker images
- **ECS** (Elastic Container Service) for running containers
- **CodeBuild** for building Docker images
- **CodePipeline** for continuous integration and deployment

## Steps
1. Create an **ECR repository** to store your Docker images.
2. Write a **Dockerfile** for your application.
3. Create a **buildspec.yml** file to define build commands and push the image to ECR.
4. Create an **ECS cluster** and define a **task definition** with the container details.
5. Set up an **ECS service** to run and manage your containers.
6. Configure **CodePipeline** to:
   - Pull code from your repository (GitHub, Bitbucket, CodeCommit, etc.)
   - Use **CodeBuild** to build and push images to ECR
   - Deploy the new task definition to ECS automatically

## Notes
- Make sure your **IAM roles and permissions** are set up correctly for ECS, ECR, CodeBuild, and CodePipeline.
- You can use **Fargate** if you prefer serverless container hosting, or **EC2 launch type** if you need more control.
- Consider setting up **CloudWatch logs** and **ECS Autoscaling** for monitoring and scaling your application.

---
