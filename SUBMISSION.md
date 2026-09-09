# Movie Picture Pipeline — Submission

**Repository:** https://github.com/akshatzeta/movie-picture-pipeline

## Live URLs

- **Frontend:** http://ac9045068329b406bb313f84c52131db-721810241.us-east-1.elb.amazonaws.com
- **Backend API:** http://aa1dc1af7fc2646daa03cfb4b169b4a3-1724090263.us-east-1.elb.amazonaws.com/movies

The frontend fetches the movie list live from the backend via the `REACT_APP_MOVIE_API_URL` value baked into the build at deploy time, and both are running on the same EKS cluster.

## Pipeline Overview

Four GitHub Actions workflows live in `.github/workflows/`:

| Workflow | File | Trigger | Jobs |
|---|---|---|---|
| Frontend CI | `frontend-ci.yaml` | PR to `main` (frontend paths) + manual | lint, test (parallel) → build |
| Backend CI | `backend-ci.yaml` | PR to `main` (backend paths) + manual | lint, test (parallel) → build |
| Frontend CD | `frontend-cd.yaml` | push to `main` (frontend paths) + manual | lint, test → build & push to ECR (SHA-tagged) → deploy to EKS via kustomize |
| Backend CD | `backend-cd.yaml` | push to `main` (backend paths) + manual | lint, test → build & push to ECR (SHA-tagged) → deploy to EKS via kustomize |

**Infrastructure** (Terraform, in `setup/terraform/`): VPC with public/private subnets, an EKS cluster (`cluster`, Kubernetes 1.36) with one worker node group, two ECR repositories (`frontend`, `backend`), and a dedicated `github-action-user` IAM user (scoped to `eks:*`, `ecr:*`, `ec2:*`) that GitHub Actions authenticates as. That user is also registered in the cluster's `aws-auth` ConfigMap with `system:masters`, so its access is validated both at the AWS IAM layer and the Kubernetes RBAC layer.

**Credentials:** No AWS access keys or secrets appear anywhere in the workflow YAML. `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` are stored as encrypted GitHub Secrets; `AWS_REGION`, `EKS_CLUSTER_NAME`, and `REACT_APP_MOVIE_API_URL` are stored as GitHub repository Variables (non-sensitive, so visible as plain text by design).

## Issues Encountered and Fixed

1. **EKS Kubernetes version 1.25 deprecated by AWS.** Bumped to 1.36 (the current default) and upgraded the AWS provider constraint from a pinned `4.55.0` to `>= 5.40.0, < 6.0.0` to support the newer AMI type enum.
2. **AL2 AMI no longer published for modern EKS versions.** Switched the node group to `AL2023_x86_64_STANDARD` and updated the SSM parameter path accordingly.
3. **Backend lint failing (`flake8: not found`).** `pipenv install` only installs the `[packages]` group by default; changed to `pipenv install --dev` in both backend workflows so dev-only tools (flake8, black, coverage) install too.
4. **Backend Docker build failing to compile `uwsgi`.** The unpinned `python:3.10-alpine` base image resolved to Alpine 3.24, whose GCC 15 treats old-style C signature mismatches in `uwsgi`'s source as hard errors. Pinned the base image to `python:3.10-alpine3.19` (GCC 13), which builds cleanly.
5. **Frontend showing stale content after a rebuild with an unchanged commit SHA.** Because ECR repositories are mutable and the image tag is derived from `github.sha`, re-running a deploy without a new commit reused an identical tag string. Kubernetes saw no diff in the Deployment spec and never re-pulled the image, so the old pod kept serving a stale JS bundle. Fixed by adding `imagePullPolicy: Always` to both the frontend and backend Deployment manifests.

## Verification

- All CI checks (lint/test/build, frontend and backend) passed on the merged PR.
- All CD checks (lint/test/build/push/deploy, frontend and backend) passed and successfully deployed to the EKS cluster.
- Backend `/movies` endpoint confirmed returning live JSON data.
- Frontend confirmed rendering the movie list fetched from the live backend URL.

## Teardown

Infrastructure will be torn down via `terraform destroy` from `setup/terraform/` once grading is confirmed complete, to avoid unnecessary AWS resource usage.
