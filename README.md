# deployments
Testing GitHub deployments APIs

## Goal

* When a commit is pushed to a branch with an open PR, deploy that commit to an S3 bucket in a directory prefixed with the short SHA of the commit
* When a PR is merged to `master`, deploy that commit to the root of the same S3 bucket as a 'development' deployment
* When a tag is created, deploy that commit to a different S3 bucket as a 'production' deployment
* Use the GitHub [deployments API](https://docs.github.com/en/rest/guides/delivering-deployments) to describe deployments

## Infrastructure

### S3 Buckets

I created two new buckets: 

* 97a79ff-deployments-dev
* 97a79ff-deployments-prod

For each bucket, I manually configured the following:

* Enabled static website hosting, with index and error documents et to `index.html`
* Disabled the 'block public access' setting
* Set 'object ownership' to 'ACLs enabled' and 'Bucket owner preferred'
* Added 'List' and 'Read' permissions for the 'Everyone' ACL

### IAM User or Role

GitHub actions needs AWS credentials to deploy, and I chose to create a new user specifically for this task.

I attached a single policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::97a79ff-deployments-*",
                "arn:aws:s3:::97a79ff-deployments-*/*"
            ],
            "Effect": "Allow"
        }
    ]
}
```

Allowing `s3:*` is totally unnecessary, but the risk is low and I don't want to take a bunch of time to figure out what least permissions for a deployment are.  Perhaps just `ListObjects` on the bucket and `PutObject`, and `PutObjectAcl` on the contents.  I'll probably circle back on this.

## GitHub Actions

As it turns out, GitHub has made managing deployments pretty transparent.  Here's the (truncated) configuration for pull request deployments:

```yaml
  deploy-pr:
    name: Deploy Pull Request
    needs: [build]
    if: github.ref != 'refs/heads/master' && !startsWith(github.ref, 'refs/tags/')
    runs-on: ubuntu-latest
    environment: 
      name: pull request
      url: http://97a79ff-deployments-dev.s3-website-us-east-1.amazonaws.com/ui-${{ env.SHORT_SHA }}
    steps:
      - uses: actions/checkout@v2
      - run: |
          echo "SHORT_SHA=$(git rev-parse --short HEAD)" >> $GITHUB_ENV
      - uses: actions/download-artifact@v2
        ...
      - uses: aws-actions/configure-aws-credentials@v1
        ...
      - run: ... deploy to s3://97a79ff-deployments-dev/ui-${{ env.SHORT_SHA }}
```

If a value is specified for `environment`, GitHub will automatically create a deployment when the job starts, and will update it with the outcome when it finishes.