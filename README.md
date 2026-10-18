# CI/CD Release & Rollback Management with Jenkins, GitHub Tags & Docker Swarm

## Overview
This repository contains Homework 3 for Cloud Computing. It expands upon the baseline pipeline by introducing semantic version release tagging (\MAJOR.MINOR.PATCH\), tag-only CI/CD triggers, immutable container image publishing to Amazon ECR, and safe zero-downtime rollback capabilities.

## Architecture & Endpoints
- **Source Control:** https://github.com/sokhit642-mit/hw3-cloud-pipeline
- **CI/CD Server:** Jenkins Parameterized Pipeline (\hw3-release-pipeline\)
- **Container Registry:** Amazon ECR (\251528148716.dkr.ecr.ap-southeast-2.amazonaws.com/my-app-h2\)
- **Live Swarm Endpoints:**
  - Manager Node: http://32.236.227.255
  - Worker Node: http://13.210.39.69

## Documentation
- See [RELEASING.md](RELEASING.md) for full instructions on version tagging conventions, release execution, and rollback procedures.
