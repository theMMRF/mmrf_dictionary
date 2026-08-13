# MMRF dictionary release setup

The MMRF-owned repository validates every pull request and every push to
`master`. Publishing occurs only when a GitHub release with a semantic version
tag (for example, `0.0.37`) is published.

The release workflow builds `schema.json`, attaches it and its SHA-256 checksum
to the GitHub release, obtains short-lived AWS credentials through GitHub OIDC,
and uploads immutable copies to S3.

Because this repository is public and uses a standard Ubuntu runner, its GitHub
Actions usage is free under [GitHub's current billing policy][actions-billing].

## 1. Choose the public artifact location

Use an MMRF-controlled S3 bucket or an MMRF-controlled CloudFront distribution.
The Gen3 services that read `dictionaryUrl` must be able to fetch the resulting
URL without interactive AWS credentials.

For an existing bucket, choose a prefix such as `mmrf_dictionary`. A release
will be stored at:

```text
s3://mmrf-dictionary-releases/mmrf_dictionary/VERSION/schema.json
```

If the bucket is private, put CloudFront in front of it with Origin Access
Control and use the CloudFront hostname for `DICTIONARY_PUBLIC_BASE_URL`.
Avoid making unrelated objects in a shared bucket public.

If you use a directly public S3 URL instead, grant anonymous `s3:GetObject`
only for `arn:aws:s3:::mmrf-dictionary-releases/mmrf_dictionary/*` in the
bucket policy and keep all write access private. Do not grant public
`s3:ListBucket` or `s3:PutObject`.

## 2. Configure GitHub as an AWS OIDC provider

The commands below require an AWS identity that can administer IAM. First get
the AWS account ID and check whether the GitHub provider already exists:

```bash
aws sts get-caller-identity --query Account --output text
aws iam list-open-id-connect-providers
```

In the provider list, look for an ARN ending in
`oidc-provider/token.actions.githubusercontent.com`. Create it only if it is
not already present in this AWS account:

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

This repository uses GitHub's immutable OIDC subject format because it was
created after GitHub introduced immutable owner and repository IDs. Its current
OIDC subject prefix can be verified with:

```bash
gh api repos/theMMRF/mmrf_dictionary/actions/oidc/customization/sub
```

Next, save the following as `mmrf-dictionary-trust-policy.json`, replacing
`AWS_ACCOUNT_ID`. The `sub` restriction permits only release jobs from this
exact repository, identified by its immutable owner and repository IDs, that
use the `dictionary-release` environment.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::AWS_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:theMMRF@155470122/mmrf_dictionary@1332388091:environment:dictionary-release"
        }
      }
    }
  ]
}
```

Create the role from that trust policy:

```bash
aws iam create-role \
  --role-name mmrf-dictionary-release \
  --description "Publish versioned MMRF dictionary artifacts from GitHub Actions" \
  --assume-role-policy-document file://mmrf-dictionary-trust-policy.json
```

If the role already exists, replace its trust policy instead:

```bash
aws iam update-assume-role-policy \
  --role-name mmrf-dictionary-release \
  --policy-document file://mmrf-dictionary-trust-policy.json
```

Save the following as `mmrf-dictionary-s3-policy.json`. It targets the
MMRF-owned `mmrf-dictionary-releases` bucket and the default
`mmrf_dictionary` prefix. This is the role's permissions policy, separate from
its trust policy.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublishVersionedDictionaryArtifacts",
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::mmrf-dictionary-releases/mmrf_dictionary/*"
    },
    {
      "Sid": "VerifyPublishedDictionaryArtifacts",
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::mmrf-dictionary-releases/mmrf_dictionary/*"
    }
  ]
}
```

Attach the permissions policy and record the role ARN reported by the last
command:

```bash
aws iam put-role-policy \
  --role-name mmrf-dictionary-release \
  --policy-name PublishMMRFDictionaryArtifacts \
  --policy-document file://mmrf-dictionary-s3-policy.json

aws iam get-role \
  --role-name mmrf-dictionary-release \
  --query Role.Arn \
  --output text
```

No AWS access key or secret access key is stored in GitHub.
This follows [GitHub's AWS OIDC guidance][github-oidc] and scopes the AWS trust
policy to this repository's protected release environment.

## 3. Configure the GitHub environment and variables

In `theMMRF/mmrf_dictionary`, open **Settings → Environments → New
environment**, enter `dictionary-release`, and choose **Configure environment**.

Under **Deployment branches and tags**, choose **Selected branches and tags**
and add the tag pattern `*.*.*`. The workflow separately enforces an exact
numeric `MAJOR.MINOR.PATCH` format.

Do not configure **Required reviewers** if publishing a GitHub release is the
MMRF's release approval event. In that model, restrict permission to create
releases and semantic-version tags to the appropriate repository maintainers,
and require pull-request review before changes reach `master`.

Required environment reviewers may be enabled later if the MMRF decides that
publishing to S3 needs a second approval beyond the reviewed merge and release
creation. Enabling that rule causes every release workflow to wait for a
deployment approval. The AWS trust policy still limits role assumption to jobs
from this repository using the `dictionary-release` environment, but without a
required reviewer, an authorized repository maintainer can initiate publishing
without a second person approving the deployment.

On that environment page, use **Environment variables → Add environment
variable** to configure the following values. These are identifiers rather
than credentials, so they do not need to be stored as secrets.

| Variable | Required | Example |
| --- | --- | --- |
| `DICTIONARY_RELEASE_ROLE_ARN` | Yes | `arn:aws:iam::123456789012:role/mmrf-dictionary-release` |
| `DICTIONARY_AWS_REGION` | Yes | `us-east-1` |
| `DICTIONARY_ARTIFACT_BUCKET` | Yes | `mmrf-dictionary-releases` |
| `DICTIONARY_ARTIFACT_PREFIX` | No | `mmrf_dictionary` |
| `DICTIONARY_PUBLIC_BASE_URL` | No | `https://dictionary.themmrf.org` |

The public base URL must not include the dictionary prefix or version. If it is
unset, the workflow reports the standard virtual-hosted S3 URL.

Before publishing, open **Settings → Actions → General** and confirm that
GitHub Actions are enabled for the repository. The workflow declares the
minimal token permissions it needs, including `id-token: write` for OIDC and
`contents: write` for attaching files to a release.

## 4. Publish a release

1. Merge a validated dictionary change to `master`.
2. Create a new semantic-version tag following the repository's versioning
   policy.
3. Publish a GitHub release for that tag.
4. Wait for **Test and release MMRF dictionary** to finish.
5. Open the reported `schema.json` URL without AWS credentials.
6. Update the target environment's `dictionaryUrl` in `mmrf_gen3` to the new,
   immutable versioned URL.

You can perform an anonymous reachability check with:

```bash
curl --fail --location --silent --show-error \
  "PUBLIC_BASE_URL/mmrf_dictionary/VERSION/schema.json" >/dev/null
```

Do not overwrite a published version. If an artifact is incorrect, create a
new patch release.

## Why GitHub Actions instead of CodePipeline

The source and release already live in GitHub, and this workflow requires only
one short build plus a narrowly scoped S3 upload. OIDC gives the job temporary
AWS credentials and avoids managing a webhook, CodeStar connection, artifact
bucket, CodeBuild project, and long-lived credentials for a second pipeline.
CodePipeline remains appropriate for broader infrastructure deployments, but
adds no useful isolation for this small artifact-publishing task.

[actions-billing]: https://docs.github.com/en/billing/concepts/product-billing/github-actions
[github-oidc]: https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws
