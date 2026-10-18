# Release and Rollback Strategy (Semantic Versioning)

## 1. Versioning Convention
This project adheres to Semantic Versioning (SemVer) formatted as \MAJOR.MINOR.PATCH\:
- **MAJOR (\X.0.0\):** Breaking changes or major architectural overhauls.
- **MINOR (\1.X.0\):** New features or visual updates added in a backwards-compatible manner.
- **PATCH (\1.0.X\):** Backwards-compatible bug fixes or styling corrections.

## 2. Release Procedure (Git Tag Triggered)
Deployments are never triggered by ordinary branch commits. They are triggered strictly by pushing Git tags:

1. Commit application changes to \main\:
   \\\ash
   git add .
   git commit -m "feat: release version 1.0.0"
   git push origin main
   \\\
2. Create an annotated Git tag and push it to GitHub:
   \\\ash
   git tag -a v1.0.0 -m "Release version 1.0.0"
   git push origin v1.0.0
   \\\
3. Jenkins detects the tag push via webhook/polling, builds the Docker image tagged with \:v1.0.0\, pushes it to Amazon ECR, and executes a zero-downtime rolling update on Docker Swarm.

## 3. Rollback Procedure
If a defect is detected in a live release:
1. Open Jenkins: \http://localhost:8080/job/hw3-release-pipeline/\.
2. Click **Build with Parameters**.
3. Select parameters:
   - **ACTION:** \ollback\
   - **VERSION:** Enter the previously verified stable tag (e.g., \1.0.0\).
4. Click **Build**.
5. Jenkins verifies the image tag exists in Amazon ECR and instructs the Swarm Manager to redeploy that image tag directly without rebuilding anything.
