# PT-LIMS GitHub Actions to ECR POC

This package performs one job only: build a tiny proof-of-concept container image in GitHub Actions and push it to the existing PT-LIMS External ECR repository.

It does not contain database code, RDS configuration, Secrets Manager references, ECS task definitions, deployment steps, application dependencies, or credentials.

## Confirmed configuration

| Setting | Value |
| --- | --- |
| GitHub repository | `DEFRA/apha-ptl-apps` |
| AWS account | `783559591987` |
| AWS region | `eu-west-2` |
| ECR repository | `apha/ptlexternal` |
| ECR URI | `783559591987.dkr.ecr.eu-west-2.amazonaws.com/apha/ptlexternal` |
| GitHub OIDC role | `arn:aws:iam::783559591987:role/github_actions_pipeline_ecr_role` |
| OIDC audience | `sts.amazonaws.com` |

These non-secret values are configured directly in the workflow. Do not create GitHub secrets containing AWS access keys.

## Package contents

```text
.github/workflows/ptlims-ecr-push-poc.yml
poc/ecr-push/Dockerfile
poc/ecr-push/.dockerignore
poc/ecr-push/ecr-poc.txt
iam/github-ecr-push-policy-reference.json
iam/github-oidc-trust-policy-reference.json
README.md
```

The image uses `FROM scratch` and contains only `ecr-poc.txt`. It does not pull a base image from Docker Hub or another registry. It is intentionally non-runnable because this POC validates image build and ECR publishing only.

## Step 1: Confirm the AWS configuration

In AWS Console, ask CCoE or an authorised user to confirm:

1. Open **IAM > Roles > github_actions_pipeline_ecr_role**.
2. Under **Trust relationships**, confirm the federated principal is `token.actions.githubusercontent.com`.
3. Confirm the audience condition is `sts.amazonaws.com`.
4. Confirm the subject condition allows `repo:DEFRA/apha-ptl-apps:ref:refs/heads/main`.
5. Under **Permissions**, confirm the role can obtain an ECR token and push to `arn:aws:ecr:eu-west-2:783559591987:repository/apha/ptlexternal`.
6. Open **Amazon ECR > Private registry > Repositories > apha/ptlexternal** and confirm the repository exists.

The JSON files under `iam/` are review references. Do not replace CCoE-managed policies without their approval.

## Step 2: Add the files to GitHub

1. Extract this ZIP.
2. Open the extracted `ptlims-ecr-push-poc` folder in VS Code.
3. Copy its contents into the root of the local `DEFRA/apha-ptl-apps` repository. Keep the folder structure unchanged.
4. Create a feature branch, for example:

   ```bash
   git switch -c feature/ptlims-ecr-push-poc
   ```

5. Review the files and commit them:

   ```bash
   git add .github/workflows/ptlims-ecr-push-poc.yml poc/ecr-push iam README.md
   git commit -m "Add PT-LIMS ECR push POC"
   git push -u origin feature/ptlims-ecr-push-poc
   ```

6. Raise a pull request and merge it into `main` after approval. The reference trust policy is restricted to `main`, so running it from an unapproved feature branch may fail during OIDC role assumption.

## Step 3: Run the workflow

The workflow runs automatically when its POC files are merged into `main`. It can also be run manually:

1. Open the GitHub repository.
2. Select **Actions**.
3. Select **PT-LIMS ECR Push POC**.
4. Select **Run workflow**.
5. Choose the `main` branch.
6. Select **Run workflow**.

The workflow then:

1. Checks out the repository.
2. Builds the tiny local image.
3. Requests a GitHub OIDC token.
4. Assumes `github_actions_pipeline_ecr_role`.
5. Runs `aws sts get-caller-identity` and verifies account `783559591987`.
6. Logs in to ECR.
7. Pushes a uniquely tagged image to `apha/ptlexternal`.
8. Reads back the immutable image digest.
9. Writes the image URI and digest to the GitHub workflow summary.

## Step 4: Verify the result

### In GitHub

1. Open the successful workflow run.
2. Confirm **Verify assumed AWS identity** shows account `783559591987` and the assumed `github_actions_pipeline_ecr_role` session.
3. Confirm **Tag and push image** completed successfully.
4. Open the workflow summary and record the image URI and digest.

### In AWS Console

1. Open the AWS Console in account `783559591987` and region `eu-west-2`.
2. Open **Amazon ECR**.
3. Select **Private registry > Repositories**.
4. Select `apha/ptlexternal`.
5. Find the new tag beginning with `poc-`.
6. Confirm its push timestamp, digest, size, and scan status.

At this point the POC is complete. A green workflow, a new `poc-...` tag, and the matching digest in ECR prove that GitHub OIDC role assumption and ECR push permissions work.

## Common failures

| Failure | Likely cause |
| --- | --- |
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | Role trust policy does not allow the repository, branch, or `sts.amazonaws.com` audience |
| Account verification fails | The workflow assumed a role in the wrong AWS account |
| `ecr:GetAuthorizationToken` denied | Publishing role lacks registry-login permission |
| Layer upload or `ecr:PutImage` denied | Publishing role lacks repository push permissions |
| Repository not found | Incorrect region/repository name, or `apha/ptlexternal` has not been created |
| Tag already exists | ECR tag immutability rejected a reused tag; rerun creates a new tag using the run attempt |

The action references use readable version tags. Replace them with CCoE-approved full commit SHAs if required by DEFRA supply-chain policy.
