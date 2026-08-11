# Solvent shared workflows

Reusable GitHub Actions workflows and CloudFormation templates for client
projects. Client repos pin a major version tag, so upgrading a client is a
one-line diff.

## Onboarding a static-site client

1. Create the AWS account (one per client) and a Route53 hosted zone for the
   domain. Give the registrar the zone's nameservers.

2. Deploy the bootstrap stack manually, with an admin credential:

   ```sh
   aws cloudformation deploy \
     --template-file templates/bootstrap-github-oidc.yaml \
     --stack-name solvent-bootstrap \
     --region us-east-1 \
     --parameter-overrides \
       GitHubOrg=solvent-software-llc \
       RepoName=acmecorp.com \
       ClientSlug=acme-corp \
     --capabilities CAPABILITY_NAMED_IAM
   ```

   `ClientSlug` must match the `slug` in `client.yaml` — the deploy role's S3
   permissions are scoped to `<slug>-site-<account-id>`.

3. Set the stack's `RoleArn` output as the `AWS_ROLE_ARN` repository variable
   in the client repo.

4. Copy `examples/client.yaml`, `examples/deploy.yml`, and
   `examples/validate.yml` into the client repo. `client.yaml` goes at the repo
   root; the two workflows go in `.github/workflows/`.

5. Push to `main`.

The first deploy blocks on ACM DNS validation and takes 15–25 minutes, mostly
CloudFront propagation. Later deploys are 2–3 minutes.

## client.yaml

| Key | Required | Notes |
|---|---|---|
| `slug` | yes | Resource-naming token. Lowercase, hyphens, 3–32 chars. |
| `name` | no | Human-readable name for AWS tags. Defaults to `slug`. |
| `tier` | no | Must be `static-site` for these workflows. |
| `domain` | yes | Apex or subdomain FQDN. |
| `www` | no | Defaults to `true`. Set `false` for subdomain sites. |
| `aws.region` | yes | Must be `us-east-1` for the static tier. |
| `aws.stack_name` | yes | CloudFormation stack name. |
| `aws.hosted_zone_id` | yes | Route53 zone containing `domain`. |
| `aws.bucket_name_override` | no | Only for stacks predating the naming convention. |
| `public_env` | no | `PUBLIC_*` keys written to `.env` at build time. |

Everything in `public_env` ships in the client bundle. It is public by
definition — never put secrets there.

## Versioning

Templates and workflows are released together under one tag. `v1` is a moving
major tag; point Renovate at it, or pin exact tags per client for manual
control over upgrades.

The deploy workflow checks this repo out at `GITHUB_JOB_WORKFLOW_REF`, so the
CloudFormation template a client uses is always the one released alongside the
workflow they pinned.

## Requirements on the client repo

- `npm run check`, `npm run lint`, and `npm run build` all exist
- Build output at `./build`, with hashed assets under `./build/_app/immutable`
- `client.yaml` at the repo root
- `AWS_ROLE_ARN` set as a repository variable
