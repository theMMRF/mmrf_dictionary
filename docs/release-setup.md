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
s3://BUCKET/mmrf_dictionary/VERSION/schema.json
```

If the bucket is private, put CloudFront in front of it with Origin Access
Control and use the CloudFront hostname for `DICTIONARY_PUBLIC_BASE_URL`.
Avoid making unrelated objects in a shared bucket public.

If you use a directly public S3 URL instead, grant anonymous `s3:GetObject`
only for `arn:aws:s3:::BUCKET/mmrf_dictionary/*` in the bucket policy and keep
all write access private. Do not grant public `s3:ListBucket` or `s3:PutObject`.

## 2. Configure GitHub as an AWS OIDC provider

Create the provider once per AWS account if it does not already exist:

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

Create an IAM role with the following trust policy, replacing
`AWS_ACCOUNT_ID`. The `sub` restriction permits only release jobs from this
repository that use the protected `dictionary-release` environment.

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
          "token.actions.githubusercontent.com:sub": "repo:theMMRF/mmrf_dictionary:environment:dictionary-release"
        }
      }
    }
  ]
}
```

Attach only the permissions needed to publish dictionary artifacts, replacing
`BUCKET` and the prefix if necessary:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublishVersionedDictionaryArtifacts",
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::BUCKET/mmrf_dictionary/*"
    },
    {
      "Sid": "VerifyPublishedDictionaryArtifacts",
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::BUCKET/mmrf_dictionary/*"
    }
  ]
}
```

No AWS access key or secret access key is stored in GitHub.
This follows [GitHub's AWS OIDC guidance][github-oidc] and scopes the AWS trust
policy to this repository's protected release environment.

## 3. Configure the GitHub environment and variables

In repository settings, create an environment named `dictionary-release`.
Restrict its deployment tags to a semantic-version pattern and add required
reviewers if desired. Configure these environment variables:

| Variable | Required | Example |
| --- | --- | --- |
| `DICTIONARY_RELEASE_ROLE_ARN` | Yes | `arn:aws:iam::123456789012:role/mmrf-dictionary-release` |
| `DICTIONARY_AWS_REGION` | Yes | `us-east-1` |
| `DICTIONARY_ARTIFACT_BUCKET` | Yes | `mmrf-dictionary-artifacts` |
| `DICTIONARY_ARTIFACT_PREFIX` | No | `mmrf_dictionary` |
| `DICTIONARY_PUBLIC_BASE_URL` | No | `https://dictionary.themmrf.org` |

The public base URL must not include the dictionary prefix or version. If it is
unset, the workflow reports the standard virtual-hosted S3 URL.

## 4. Publish a release

1. Merge a validated dictionary change to `master`.
2. Create a new semantic-version tag following the repository's versioning
   policy.
3. Publish a GitHub release for that tag.
4. Wait for **Test and release MMRF dictionary** to finish.
5. Open the reported `schema.json` URL without AWS credentials.
6. Update the target environment's `dictionaryUrl` in `mmrf_gen3` to the new,
   immutable versioned URL.

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
