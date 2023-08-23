---
layout: post
title: "Your OIDC Deployments From Public Repositories May Be At Risk"
description: "Your AWS account may be at risk if you use OIDC to deploy resources from public GitHub repositories. Lean how to block outside collaborators and restrict IAM permissions."
backgroundUrl: "https://images.unsplash.com/photo-1585433341185-9e60b28ed0a7?ixlib=rb-1.2.1&auto=format&q=80&fit=crop"
---

> tl;dr: Your AWS account may be at risk if you use OIDC to deploy resources from a public GitHub repository.
> Use GitHub's settings for GitHub Actions to block outside collaborators, or add a condition to your IAM policy to only allow a certain GitHub user.

Remember the days when you had to use long-lived credentials that could get into the wrong hands?
Those days are thankfully behind us. Today, OpenID Connect (OIDC) is the best way for
Continuous Integration (CI) pipelines to assume a short-lived session in your AWS account.

If you follow some of the guides out there for your public repositories you may be at a high risk though. Anyone
who can create a pull request may be able to access resources in your account, and even incur heavy spending by spinning up expensive compute.

In this article I'll show you how to get up to speed with OIDC, how it can be abused, and how you can secure your public
repositories. This problem doesn't affect private repositories as much, since there's no public access.

## What is OIDC and How Do I Use it?

OIDC is one of the many ways your GitHub workflow can assume a role with permissions in your AWS account.
There are many more ways you can achieve that, or what OIDC can do, but for the scope of this article that's enough.
[You can still read up on OIDC](https://openid.net/developers/how-connect-works/).

The gist is that you set up an OIDC config in your AWS account, and then your GitHub Actions (GHA) workflow can assume a role in your account.

When it comes to setting up the AWS configuration and the GHA workflows that deploy resources into your AWS account, here are two good resources:

* [This AWS blog post will guide you through the steps needed to configure a GitHub workflow to assume a role in your AWS account to perform changes.](https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/)
* [This GitHub guide goes deeper into the details of the workflow and how you should configure it.](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)

Amongst others, you'll have a GitHub workflow that has a step similar to this somewhere:

```yaml
  - name: configure aws credentials
    uses: aws-actions/configure-aws-credentials@v2
    with:
      role-to-assume: arn:aws:iam::1234567890:role/example-role
      role-session-name: samplerolesession
      aws-region: us-east-1
```

There's a big problem that the articles don't call out though: *Any GitHub user with access to your repository can run arbitrary
workflows*, unless you restrict which GitHub users can start workflows, and secure your AWS account. Workflows can be added
in pull requests and will run automatically.

In the example below you only need to create a pull request that contains a new workflow which runs on the `pull_request` event:

```yaml
on:
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - name: Run tests
      run: echo "I'm a new workflow..."
```

## Fix It

There are two options to fix the problem:

1. The first uses GitHub settings that can be activated with a few clicks.
2. The second one uses IAM Conditions to add a restriction from AWS' side.

Personally I find the second one more secure, because it uses a mechanism that is unlikely to change (i.e. needs a
breaking API change) and is well documented.
However only the first seems to be viable for organizations, since I haven't found a way to get the
GitHub organization name of a user (not the repo!) from the OIDC token.

## Option 1: Block Outside Collaborators in GitHub

What are outside collaborators?

[GitHub explains this as following](https://docs.github.com/en/organizations/managing-user-access-to-your-organizations-repositories/adding-outside-collaborators-to-repositories-in-your-organization):

> An outside collaborator is a person who is not a member of your organization, but has access to one or more of your organization's repositories.

The easier option is to click a radio box in your repository's settings. Go to https://github.com/<YOUR_USERNAME>/<YOUR_REPOSITORY>/settings/actions,
click on "Require approval for all outside collaborators", and click on Save.

![Downloader Pattern](https://bahr.dev/pictures/deny-outside-collaborators.png)

People who are not part of your GitHub organization should now not be able to start workflows with pull requests in your repository,
and therefore not be able to assume credentials for your AWS account via OIDC. I guess that for repositories without a GitHub organization,
this means that only you can start workflows. I have not found further documentation on what outside collaborators mean without
organizations. I assume that for personal repositories this means if you invited a collaborator or not. If you know more please let me know.

Unfortunately that means you can't invite bots to your repository for [automating your dependency upgrades](https://bahr.dev/2022/12/05/automatic-depdency-upgrades/).
Bots end with a `[bot]` in their username. `dependabot-bot` is not the bot user `dependabot[bot]`. Be careful with those!

## Option 2: Harden the IAM Policy

The IAM policies described in the articles I linked above can be further restricted with a condition for a GitHub organization or username.

Before adding the condition the IAM policy may look like this example:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
              "Federated": "arn:aws:iam::123456123456:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
              "StringEquals": {
                "token.actions.githubusercontent.com:sub": "repo:bahrmichael/my-repo:*",
                "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
              }
            }
        }
    ]
}
```

You can see that there already are some conditions, e.g. for the repository.

The [OIDC token contains amongst others a field for the `actor`](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#understanding-the-oidc-token). That's our GitHub user.

To allow only our own GitHub user (in my case `bahrmichael`) we can add the condition `"token.actions.githubusercontent.com:actor": "bahrmichael"`.

The result may look like below:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::123456123456:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:sub": "repo:bahrmichael/my-repo:*",
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
                    "token.actions.githubusercontent.com:actor": "bahrmichael"
                }
            }
        }
    ]
}
```

This approach allows anyone to create a pull request, but only pull requests created by `bahrmichael` are allowed to assume the role and therefore deploy into the AWS account.

## Test It

To test this solution, use a GitHub user who doesn't have permission to deploy.
Try creating a pull request that triggers a deployment to your AWS account (or anything that would attempt to assume the role)
and watch it fail. This is a clear indication that your safeguard is working!

## Conclusion

Congratulations! By following this guide you blocked anonymous malicious actors from accessing your AWS account via OIDC roles in GHA workflows.

Unfortunately this safeguard doesn't work well with dependency automation. If you want to keep your dependencies up to date,
then a private repository may be a better approach.

If you have any questions or feedback, please raise them in [GitHub](https://github.com/bahrmichael/bahrmichael.github.io).

## Resources

- [Secretless connections from GitHub Actions to AWS using OIDC](https://blog.codecentric.de/secretless-connections-from-github-actions-to-aws-using-oidc)
- [Secure AWS deploys from GitHub Actions with OIDC](https://www.eliasbrange.dev/posts/secure-aws-deploys-from-github-actions-with-oidc)
- [Use IAM roles to connect GitHub Actions to actions in AWS](https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/)
- [How does OIDC work?](https://openid.net/developers/how-connect-works/)
- [Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [Understanding the OIDC token](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#understanding-the-oidc-token)
- [Keeping your GitHub Actions and workflows secure Part 1: Preventing pwn requests](https://securitylab.github.com/research/github-actions-preventing-pwn-requests/)
- [Keeping your GitHub Actions and workflows secure Part 2: Untrusted input](https://securitylab.github.com/research/github-actions-untrusted-input/)
- [Keeping your GitHub Actions and workflows secure Part 3: How to trust your building blocks](https://securitylab.github.com/research/github-actions-building-blocks/)
