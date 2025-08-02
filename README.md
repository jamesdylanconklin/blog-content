# Blog Content and Deploy Pipeline

Repository for creating and storing Hugo Blog content and publishing to S3 on PR merge.

## Content

Vanilla Hugo. Follow the [Quick-Start Guide](https://gohugo.io/getting-started/quick-start/) Boutique theming and other cosmetic concerns are a later nice-to-have.

## Deploy

We will define a GitHub Action that, on PR merge and on Tagged release:
- Pulls some config pupulated to AWS Parameter Store by the blog-infra project
  - IAM role for GH Actions (Using bootstrapped, minimal role in GH Action secrets)
  - domain name for Hugo config.
  - CloudFront Distribution id for cache invalidation on publish
  - Blog content S3 bucket name
- Builds the site, omitting draft or future content for Release builds
- Pushes built content to S3
- Invalidates CloudFront cache
  - potential nice-to-have: only for files changed in PR? Might get messy with invalidation costs.
- Curls a release version page from deployed site to verify successful push and cache invalidation.
