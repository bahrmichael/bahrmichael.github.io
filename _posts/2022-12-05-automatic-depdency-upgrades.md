---
layout: post
title: "Automatic Dependency Upgrades on GitHub"
description: "Stop the tedious work! Learn how to automate your dependency upgrades on GitHub."
backgroundUrl: "https://images.unsplash.com/photo-1485827404703-89b55fcc595e?ixlib=rb-1.2.1&auto=format&q=80&fit=crop"
---

Either you automate your dependency upgrades, or you wait until absolutely necessary. Everything else is just tedious.

At Stedi we recently automated all dependency upgrades across more than 90 repositories.
This lead to more than 10,000 fully automated changes within a few months. This degree of automation may sound scary,
but there are small iterative steps that you can take. Check out the later chapter [Gaining Confidence](#7-gaining-confidence-for-full-automation).

This article will help you to set up automatic dependency upgrades. We're going to look at an NPM project on GitHub, but the concepts apply to other languages as well.

If you want to hear Thorsten Höger and me discussing this on a podcast, check out the [episode #6 of Cloud Automation Weekly](https://podcast.taimos.de/episodes/06-michael-bahr).

## 1. How does the setup look like?

We will use [Renovate](https://github.com/renovatebot/renovate) and [Dependabot](https://github.blog/2020-06-01-keep-all-your-packages-up-to-date-with-dependabot/) to create pull requests. [Mergify](https://mergify.com/) will check, approve and merge the pull requests.
Your mileage may vary if you host your code outside of GitHub.

![A diagram showing that Renovate and Dependabot create pull requests, and Mergify approves and merges them.](https://bahr.dev/pictures/automatic-dependency-upgrades-overview.png)

The major driver of dependency upgrades is Renovate. It offers a high degree of configuration.
For example, you can create rules that all packages should have been released for a few weeks before upgrading them in your codebase.
You can also tell it to group upgrades that belong together, like jest and ts-jest.
From my experience you can expect Renovate to run every couple hours, but don't expect it to react within minutes.

Dependabot always creates individual pull requests for each upgrade immediately.
This is not desirable if you want to wait after a release happened, e.g. to reduce the risk of supply chain attacks.
But for security upgrades immediate pull requests are great. Renovate does that too, but we want more data sources and faster pull requests for security upgrades.

Here's an overview of what we'll configure in this article:

- [GitHub repository settings for branch protection rules and labels](#3-github-repository-settings)
- [Renovate to create pull requests](#4-configure-renovate)
- [Dependabot to create security pull requests](#5-configure-dependabot)
- [Mergify to approve and merge pull requests](#6-configure-mergify)

Let's address a common concern first.

## 2. But engineers must review every pull request!

When you get a pull request for a dependency upgrade, do you just check that the version upgrade was correct?
Or do you go further and check the dependency's code change and its dependencies' code changes?

If you only check the changes in the pull request, you may quickly discover that this work is very repetitive and is a good candidate for automation.
If you do the latter, you hopefully work at a large company that has enough resources for a team that reviews external dependencies.

Having two bots work together is actually compliant under [SOC2](https://cloud.google.com/security/compliance/soc-2), because it requires two actors.
Those actors don't have to be humans, but can also be bots. One bot can create a pull request, and another one can approve and merge it.

If you don't trust one bot or the other, you can use [GitHub's CODEOWNERS file](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) and Mergify's rules to restrict which files bots can change.

With that aside, let's dive into all the things we can configure to achieve full automation.

## 3. GitHub Repository Settings

Before we invite the bots to automate changes in our repository, we will set up some guardrails.
These guard rails are [branch protection rules](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/defining-the-mergeability-of-pull-requests/about-protected-branches) in GitHub.
They prevent the merge of changes which do not meet a given set of rules.

We will also create labels to improve the communication between Renovate and Mergify.

### 3.1. Branch Protection Rules

The easiest way to protect your main branch from breaking is to set up branch protection rules.

To configure them, go to your repository's settings, to Branches, and the add or edit the branch protection rules of your main branch.

We will set up two rules that prevent Mergify from merging a pull request prematurely. They're also great for pull requests from humans.

![The branch protection setting to require at least one approval](https://bahr.dev/pictures/automatic-dependency-upgrades-approval-rules.png)

This setting requires that a pull request gets at least one approval before the author can merge it. You may further harden the rules by requiring a review from code owners, and approval from someone else than the author.

![The branch protection setting to have the deploy check pass](https://bahr.dev/pictures/automatic-dependency-upgrades-branch-checks.png)

This setting requires that the job `deploy` from a GitHub Actions workflow has been successful. You currently only see a deployment that should succeed, but it will also include unit tests and e2e tests in the future.

### 3.2. Labels

Over the course of this article we'll use two new labels "major-upgrade" and "security". You have to enable issues in your GitHub project for this.

Navigate to `https://github.com/<your-org>/<your-repo>/issues/labels` and add two labels `major-upgrade` and `security`.

![Two labels in GitHub with the names major-upgrades and security](https://bahr.dev/pictures/automatic-dependency-upgrades-labels.png)

## 4. Configure Renovate

Renovate will be our main work horse to create pull requests for dependency upgrades.
Go through [part 1 of Renovate's tutorial](https://github.com/renovatebot/tutorial) to install their bot. After that we modify the setup pull request together.

When you grant Renovate access to your repository, it will create [a "Configure Renovate" pull request](https://github.com/bahrmichael/trade-game-backend/pull/6).
We're going to extend this configuration. Every part shown below can be picked or omitted based on your preferences.

### 4.1. Stability Days

With setting [`stabilityDays`](https://docs.renovatebot.com/configuration-options/#stabilitydays) to 21 days, we tell Renovate to only create a pull request when a release is at least 21 days old.

```json
"stabilityDays": 21
```

This helps with two problematic cases:

1. A package can be un-published from NPM for 3 days after its release.
2. In [supply chain attacks](https://learn.microsoft.com/en-us/microsoft-365/security/intelligence/supply-chain-malware) an adversary may release a package with malicious code. Unless you don't use dependencies, I don't think you can be 100% safe from supply chain attacks. Malicious release are more likely to be discovered by the community when you wait for a few weeks before upgrading.

### 4.2. Dependency Dashboard

Let Renovate create an issue in our repository that acts as a [dependency dashboard](https://docs.renovatebot.com/configuration-options/#dependencydashboard). Renovate will update this issue when necessary.

```json
"dependencyDashboard": true
```

The dependency dashboard will show you issues that have not reached the stability days, as well as ignored or blocked upgrades.

![A dependency dashboard issue showing the status of various dependency upgrades](https://bahr.dev/pictures/automated-dependency-upgrades-dependency-dashboard.png)

### 4.3. Flag Major Upgrades

Major upgrades are likely to contain breaking changes, where it makes sense for an engineer to check the changelog.

[According to semantic versioning](https://semver.org/) major upgrades contain breaking changes. In these cases it's likely that an engineer needs to help.
Therefore, we tell Renovate to not open PRs automatically, but only when an engineer triggers one via the [dependency dashboard](https://docs.renovatebot.com/configuration-options/#dependencydashboardapproval).

Furthermore, we add a label for Mergify so that it can handle major upgrades differently.

```json
"major": {
  "dependencyDashboardApproval": true,
  "addLabels": [
    "major-upgrade"
  ]
},
```

Finally, we use [`packageRules`](https://docs.renovatebot.com/configuration-options/#packagerules) to treat 0.x.x packages with upgrades of their minor version like major upgrades.

```json
"packageRules": [
  {
      "matchUpdateTypes": [
          "minor"
      ],
      "matchCurrentVersion": "/^[~^]?0/",
      "dependencyDashboardApproval": true,
      "addLabels": [
          "major-upgrade"
      ]
  }
]
```

You can extend this pattern to also treat 0.0.x upgrades like major upgrades.

You may soon notice that manually triggering dependency upgrades becomes tedious. Once you have confidence in your
setup you can remove the flag `dependencyDashboardApproval`. Renovate will then create pull requests, but Mergify
won't merge them yet. Your engineers can still review and decide if their code is ready for the upgrade.

### 4.4. Labels for Security Upgrades

For visibility, we add a [security](https://docs.renovatebot.com/configuration-options/#vulnerabilityalerts) label on pull requests that fix vulnerabilities.

```json
"vulnerabilityAlerts": {
  "addLabels": [
      "security"
  ]
}
```

### 4.5. Pin Dependencies

By default, Renovate will use ranges for dependencies. However, there are a couple [benefits to pinning dependency versions](https://docs.renovatebot.com/dependency-pinning/).

To let Renovate pin your dependencies, [set the `rangeStrategy` to `pin`](https://github.com/bahrmichael/trade-game-backend/commit/1da9b2e27e2b26322bdd75397b1c0a79cf63cbab).

```json
"rangeStrategy": "pin"
```

After you enable dependency pinning, Renovate will create a pull request that updates dependencies from ranges to pinned versions.
Here's an example [pull request](https://github.com/bahrmichael/trade-game-backend/pull/23).

Renovate doesn't automatically pin the versions of GitHub Actions. To have it pin those we add a [preset](https://docs.renovatebot.com/presets-helpers/#helperspingithubactiondigests).

```json
"extends": [ "config:base", "helpers:pinGitHubActionDigests" ]
```

### 4.6. Security Upgrades for Transitive Dependencies

With [`transitiveRemediation`](https://docs.renovatebot.com/configuration-options/#transitiveremediation) Renovate will
go a step further and try to keep your dependencies vulnerability-free by also check the dependencies of your dependencies.

```json
"transitiveRemediation": true
```

### 4.7. Finish the Renovate Setup

Once you're done updating the config in Renovate's setup pull request, merge it into your main branch.
After a while you should receive new pull requests, unless all dependencies are already pinned and up to date.

While you wait for those, you can continue with setting up Dependabot and Mergify.

## 5. Configure Dependabot

We're going to use Dependabot for security upgrades only. Configure it by creating a file `.github/dependabot.yml` with the following content:

```yaml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      # Security updates ignore this and are opened ad-hoc,
      # but this is still a required option.
      interval: daily
    # Prevent PRs from opening except for security updates
    open-pull-requests-limit: 0
```

When Dependabot detects a vulnerable dependency in your manifest file it will now send pull requests.

## 6. Configure Mergify

Mergify is a bot that can take action on existing pull requests, like approving or merging them.
It can also do [much more](https://docs.mergify.com/actions/).

I recommend using [Mergify's config editor](https://dashboard.mergify.com/) to validate your Mergify configuration.

We're going to create a `.mergify.yml` config with rules to approve and merge pull requests that come from Dependabot and Renovate.
Below you can see the full config. I'm going to explain the parts step by step.

```yaml
pull_request_rules:
  - name: Auto-approve upgrades
    conditions:
      - -label=major-upgrade
      - or:
        - author=dependabot[bot]
        - and:
            - author=renovate[bot]
            - or:
                - title~=update .* digest to \w+
                - title~=update .* monorepo
                - title~=update dependency
                - title~=update .* action to \w+
                - title~=pin .* action to \w+
    actions:
      review:
        type: APPROVE
        message: Automatically approving non-major dependency upgrade
  - name: Auto-merge upgrades
    conditions:
      - check-success=deploy
    actions:
      merge:
        method: squash
```

With this file we're setting up rules for pull requests. Each rule consists of a name, conditions, and actions.
If a pull request matches the rules, Mergify will perform the specified actions.

The first rule "Auto-approve upgrades" tells Mergify to automatically approve pull requests. The conditions specify that the pull request
1. must not be a major-upgrade,
2. must come from either Dependabot or Renovate, and
3. if it comes from Renovate, its title must match one of the patterns.

You may update the patterns as you see pull requests that you'd like to have auto-merged. If you're wondering why
Mergify does not take action on a pull request then check out the result of the action that Mergify runs on your pull request.

The second rule "Auto-merge upgrades" tells Mergify to auto-merge a pull request when

1. the branch protection rules are met, and
2. the check `deploy` is successful.

We previously defined the branch protection rules to require the `deploy` check to pass.
We reiterate it here, because Mergify might try to merge the pull request so fast that GitHub has not realised it should apply checks.

Once set up, you should see Mergify take action on pull requests. In my experience Mergify will do so in less than a minute.
If it doesn't, have a look at the checks where Mergify shows how it evaluated a pull request.

![GitHub log showing that Mergify approved and merge a pull request](https://bahr.dev/pictures/automatic-dependency-upgrades-automated-pr.png)

Congratulations, you automated your first dependency upgrade! Most of your dependencies will now automatically stay up to date.

You still need to check in from time to time if any upgrades break your tests, or if you need to help with any major upgrades.
However, that's much less work than taking care of all the small upgrades as well. To get notified about failed
dependency upgrades you can use actions for Slack or your chat tool of choice.

## 7. Gaining Confidence for Full Automation

Going for full automation right away may be scary. Luckily you can add pieces from this article over time to increase the confidence in your automation.

If you automatically deploy into production, I highly suggest that you start with writing plenty of tests first.
Make sure you have enough test coverage that you're comfortable with unsupervised deployments.

But even if you don't have great test coverage yet, you can start with letting Renovate create pull requests.
Renovate will only create them, but not merge them yet if you used the configuration above. Do this for a while and see if anything breaks.
If nothing breaks, you're all set for part-time automation.

With part-time automation, you restrict Mergify to only work on a certain schedule. This schedule could be your core working hours.

Once you gained enough trust you can extend the schedule or get rid of it altogether.

It's fine to stop at any level of automation and say that's as far as you'll go. A little bit of automation is better than nothing.

## 8. Conclusion

In this article we set up automatic dependency upgrades in a GitHub repository.
We've also seen ways to adjust the configuration to our needs, and a path to increase confidence before going for 100% automation.

A big thank you to all the developers behind Renovate, Mergify, and Dependabot for giving us such great tools, all free of charge.

You can find an example configuration for [Dependabot](https://github.com/bahrmichael/trade-game-backend/blob/main/.github/dependabot.yml),
[Renovate](https://github.com/bahrmichael/trade-game-backend/blob/main/renovate.json), and [Mergify](https://github.com/bahrmichael/trade-game-backend/blob/main/.mergify.yml),
as well as [automated pull requests](https://github.com/bahrmichael/trade-game-backend/pull/8) in [this repository](https://github.com/bahrmichael/trade-game-backend).
