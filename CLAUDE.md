# CLAUDE.md

Shared CI/CD and infrastructure for Solvent Software client projects. This repo
is a **library, not an application** — nothing here deploys on its own. Client
repos call into it.

## Consumer model

A client repo contains only `src/`, `static/`, `client.yaml`, and two ~10-line
workflows that pin a tag here:

```yaml
jobs:
  deploy:
    uses: solvent-software-llc/workflows/.github/workflows/static-site-deploy.yml@v1
```

The called workflow checks this repo out to `.solvent/` at
`GITHUB_JOB_WORKFLOW_REF`, so the CloudFormation template a client runs is
always the one released alongside the workflow they pinned. **Changing anything
here changes every client on the next deploy.** Assume a blast radius of all
client sites.

## Layout

| Path | Role |
|---|---|
| `.github/actions/client-config/action.yml` | Parses + validates `client.yaml`, writes `.env`. Single source of truth for both workflows. |
| `.github/workflows/static-site-deploy.yml` | Reusable. Build → CFN deploy → S3 sync → invalidate. |
| `.github/workflows/static-site-validate.yml` | Reusable. Same checks, no AWS writes. Runs on client PRs. |
| `.github/workflows/ci.yml` | This repo's own tests. Not reusable. |
| `templates/static-site.yaml` | S3 + CloudFront (OAC) + ACM + Route53. |
| `templates/bootstrap-github-oidc.yaml` | Per-account, run **manually** with admin creds. Never from CI. |
| `examples/` | Copied into new client repos verbatim. |

Deploy and validate must stay behaviourally identical up to the AWS steps. If
they drift, validate green-lights things deploy rejects. `ci.yml` asserts both
use the composite action.

## Hard-won constraints

Do not "simplify" these without reading the reason.

**`yq` is mikefarah v4**, not python-yq. jq-style expressions mostly work, but
`//` treats `false` as absent — `.www // true` silently turns every `www: false`
into `true`. Read the key directly and default in shell:
`WWW=$(yq -r '.www' "$F"); [ "$WWW" = "null" ] && WWW=true`.

**Never put `${{ }}` inside a `run:` block.** Context values are spliced in
before bash parses the script. Multi-line values (`toJSON`) break it outright;
config-derived values are an injection vector. Pass through `env:` always.
`ci.yml` has a check for this.

**Bucket names must not contain dots.** CloudFront reaches S3 over HTTPS and
`*.s3.<region>.amazonaws.com` won't match a name with extra labels. Hence
`<slug>-site-<account-id>`. `bucket_name_override` exists only for stacks
predating this and must never be set on a new client.

**`.env`, not `.env.production`.** `svelte-kit sync` runs in dev mode during
`npm run check`, and Vite only loads plain `.env` in every mode. It must be
written after `npm ci` and before `npm run check`.

**Two-pass S3 sync, `--delete` on the second only.** Immutable assets are
content-hashed; deleting them strands clients holding the previous
`index.html`. They expire via bucket lifecycle instead.

**403 maps to 404 in CloudFront.** S3 origins scoped to `s3:GetObject` without
`ListBucket` return 403 for missing keys.

**The static tier is us-east-1 only** — ACM certs for CloudFront are
region-locked there.

**One OIDC provider per URL per AWS account.** The bootstrap assumes one account
per client; `CreateOidcProvider: false` is the escape hatch.

**`ClientSlug` in the bootstrap must equal `slug` in `client.yaml`** — the
deploy role's S3 permissions are scoped to `<slug>-site-<account-id>`.

## client.yaml contract

Required: `slug`, `domain`, `aws.region`, `aws.stack_name`,
`aws.hosted_zone_id`.
Optional: `name` (human-readable, for AWS tags; defaults to slug), `tier`
(asserted against the workflow), `www` (default true), `public_env`,
`aws.bucket_name_override`, `preview.subdomain`.

`preview.subdomain`, when set, enables `static-site-deploy.yml`'s
`deploy-target: preview` mode (see `examples/preview.yml`): builds land under
a `preview/` key prefix in the *same* bucket and are served from
`<subdomain>.<domain>` on the *same* CloudFront distribution/cert, routed by a
CloudFront Function (`PreviewRoutingFunction` in `static-site.yaml`) that
rewrites the request URI based on the `Host` header. No second stack, bucket,
or cert per client. First enabling it on an existing stack replaces the ACM
certificate (SAN changes require replacement) and re-validates via the
already-present Route53 DNS records — a few minutes, self-service, no manual
step. `preview.subdomain` must be a single DNS label (no dots).

`slug` matches `^[a-z0-9][a-z0-9-]{1,30}[a-z0-9]$` — it is a resource-naming
token, not a display name. `public_env` keys must be `PUBLIC_*` or SvelteKit
ignores them; everything there ships in the bundle and is public by definition.

Adding a validation rule means adding a matching fixture to the
`reject-bad-config` matrix in `ci.yml`. A validator with no negative test
passes everything.

## Verifying changes locally

```sh
pipx run cfn-lint templates/*.yaml
yq -r '.runs.steps[].run' .github/actions/client-config/action.yml | shellcheck -s bash -
bash <(curl -sSL https://raw.githubusercontent.com/rhysd/actionlint/v1.7.12/scripts/download-actionlint.bash) && ./actionlint

# exercise the action directly
yq -r '.runs.steps[0].run' .github/actions/client-config/action.yml > /tmp/p.sh
CONFIG_PATH=examples/client.yaml EXPECT_TIER=static-site GITHUB_OUTPUT=/tmp/out bash /tmp/p.sh
```

actionlint reports `github.job_workflow_ref` as undefined — that's a stale type
table, the property is real. Use the `GITHUB_JOB_WORKFLOW_REF` env var anyway.

## Releasing

`v1` is a moving major tag. After CI is green on `main`:

```sh
git tag -fa v1 -m "v1: static site tier" && git push -f origin v1
```

A stale `v1` is the worst failure mode here — clients silently run old code with
no signal. Verify `git rev-parse v1^{commit}` equals `origin/main` after
tagging. Breaking the `client.yaml` contract means `v2`, not a `v1` retag.

## Direction

The fullstack tier (Go APIs on Fargate, Postgres RDS) will be **CDK**, not raw
CloudFormation — conditional composition and cross-region cert wiring are what
CFN handles worst. CDK in TypeScript, even though the services are Go; the Go
jsii bindings are second-class.

Do not retrofit existing static sites into CDK. Logical IDs would change,
forcing distribution and certificate replacement with a DNS cutover, for no
gain. CDK and hand-written CFN stacks coexist fine in one account.
