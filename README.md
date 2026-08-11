# Solvent shared workflows

Reusable GitHub Actions workflows and CloudFormation templates for client projects.
Client repos pin a major version tag;

## Onboarding a static-site client

1. Create the AWS account (one per client) and a Route53 hosted zone for the domain.
   Give the registrar the zone's nameservers.
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

4. Set the stack's `RoleArn` output as the `AWS_ROLE_ARN` repository variable.
4. Copy `examples/client.yaml`, `examples/deploy.yml`, and `examples/validate.yml`
   into the client repo and fill in `client.yaml`.
5. Push to `main`.

The first deploy blocks on ACM DNS validation and takes 15–25 minutes, mostly
CloudFront propagation. Later deploys are 2–3 minutes.

## Versioning

Templates and workflows are released together under one tag. `v1` is a moving
major tag; point Renovate at it or pin exact tags per client if you want manual
control over upgrades.

Because the deploy workflow checks this repo out at `github.job_workflow_ref`,
the CloudFormation template a client uses is always the one released alongside
the workflow they pinned.

## Requirements on the client repo

- `npm run check`, `npm run lint`, `npm run build` must all exist
- Build output at `./build`, with hashed assets under `./build/_app/immutable`
- `client.yaml` at the repo root
