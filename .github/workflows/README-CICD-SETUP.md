# CI/CD Setup Notes

These four workflows live in `.github/workflows/` and drive the project's
CI/CD. Nothing in them hardcodes AWS credentials — everything sensitive
comes from GitHub Secrets, and everything account/cluster-specific comes
from GitHub (Actions) Variables so the same YAML works for anyone who forks
the repo.

## Files
- `frontend-ci.yaml` — lint + test (parallel) → docker build, on PRs touching `starter/frontend/**`, plus manual runs.
- `backend-ci.yaml` — same shape for `starter/backend/**`.
- `frontend-cd.yaml` — lint + test → build & push to ECR (tagged with the commit SHA) → deploy to EKS via kustomize, on pushes to `main` touching `starter/frontend/**`, plus manual runs.
- `backend-cd.yaml` — same shape for the backend.

## Required GitHub Secrets
Repo Settings → Secrets and variables → Actions → **Secrets**
| Name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access key for the `github-action-user` IAM user created in Step 4 of the setup instructions |
| `AWS_SECRET_ACCESS_KEY` | Matching secret key |

## Required GitHub Variables (optional overrides)
Repo Settings → Secrets and variables → Actions → **Variables**
| Name | Default if unset | Purpose |
|---|---|---|
| `AWS_REGION` | `us-east-1` | Region of your EKS cluster / ECR repos |
| `EKS_CLUSTER_NAME` | `cluster` | Matches the Terraform output `cluster_name` |
| `REACT_APP_MOVIE_API_URL` | *(empty)* | Set this to the backend's LoadBalancer URL once the backend Service has an external hostname, e.g. `http://a1b2c3-123456789.us-east-1.elb.amazonaws.com`, so the frontend is built pointing at the live backend |

## One-time cluster access step
Before the first CD run, make sure `github-action-user` has been added to
the cluster's `aws-auth` ConfigMap (Step 5 in the project setup notes,
`setup/init.sh`) — otherwise `kubectl apply` in the deploy jobs will fail
with a 403/Unauthorized error even though AWS credentials are valid.

## ECR repository names
The workflows assume ECR repositories named exactly `frontend` and
`backend` (matching the Terraform template). If you created them with
different names via the AWS Console, update the `ECR_REPOSITORY` env value
at the top of `frontend-cd.yaml` / `backend-cd.yaml`.
