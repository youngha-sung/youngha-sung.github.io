---
layout: post
title:  "Renovate auto approve/merge with Azure DevOps"
# sub_title: "The name says it all"
date:   2022-10-27 17:02:53 -0500
# categories: jekyll update
read_time: true
author: younghasung
image: "/assets/images/posts/2022-10-27-hero.jpg"
  # - thumbnail: "/assets/images/posts/2022-10-27-hero.jpg"
image.thumbnail: "/assets/images/posts/2022-10-27-hero.jpg"
introduction: |
    \"Automerge renovate PRs automatically, without human intervention.\"
---

## What is Renovate and how to setup with Azure DevOps

<a href="https://blog.objektkultur.de/how-to-setup-renovate-in-azure-devops-to-keep-your-project-dependencies-up-to-date/" target="_blank">This blog post</a> already has a good overview and how to setup with Azure DevOps.  

## Setting automerge and platformAutoMerge

At my company, we trigger the Renovate pipeline every week. The number of PRs created by Renovate varies by teams' project sizes, but usually, the PRs are likely to be neglected and left as un-merged (most likely due to other priorities), keeping the packages out-dated. This is when Renovate automerge comes in handy.

For those non-critical updates (minor, patch, pin or digest), we can assume the updates are safe to consume as long as the our unit tests, linter are passing and build is not breaking. 

> ⚠️ Make sure you have an additional pipeline configured (CI pipeline) to run all the unit tests, linter and run build, and to run whenever a new PR is raised.

For setting automerge, please refer to <a href="https://docs.renovatebot.com/key-concepts/automerge/#configuration-examples" target="_blank">Automerge: Configuration examples</a>

Once you have configured for `automerge`, we can even speed up the merging process by configuring `platformAutoMerge`. Simply add `platformAutoMerge: true` to any configuration you added `automerge`. <a href="https://docs.renovatebot.com/configuration-options/#platformautomerge" target="_blank">Renovate reference</a>

Now if you run Renovate, it will raise PR with "Auto-complete" enabled. 

![2022-10-27-azure-auto-complete.png](/assets/images/posts/2022-10-27-azure-auto-complete.png)

## Setting auto approval

Even though we have setup the `automerge` and `platformAutoMerge`, it doesn't quite work yet because we require minimum of 2 approvals from reviewers.  

To fix this issue, we are going to use <a href="https://learn.microsoft.com/en-us/rest/api/azure/devops/git/pull-request-reviewers/create-pull-request-reviewer?view=azure-devops-rest-6.0&tabs=HTTP" target="_blank">Azure API</a> in the pipeline to update reviewers' `vote` status. 

Keep in mind, don't want to approve all the PRs, we want to auto approve the PRs associated with non-major versions only. To identify those PRs, we will do a little work around here. We will add `azureAutoApprove` to the rules that we configured `automerge`. This will mark the PR requester's `vote` status to 10 ('approved'). Now in the pipeline, we will check the requester's `vote` status of the PR and update the other reviewers' status to same value. 

### Set azureAutoApprove

In your `renovate.json` config file, add `azureAutoApprove: true` to everywhere you have `automerge`. Once this is setup and run renovate, azure will automatically mark the requester's `vote` status to approved.

Here's the snippet of renovate.json with `azureAUtoApprove`:

```json
{% raw %}
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:js-lib",
    ":disableDependencyDashboard",
    "group:test",
    "group:testNonMajor",
    "group:definitelyTyped",
    ":prHourlyLimitNone",
    ":prConcurrentLimitNone",
    ":semanticCommits",
    ":semanticCommitTypeAll(chore)"
  ],
  "azureWorkItemId": 47508,
  "stabilityDays": 5,
  "prCreation": "not-pending",
  "lockFileMaintenance": {
    "enabled": true,
    "schedule": [
      "on monday"
    ],
    "automerge": true,
    "platformAutomerge": true,
    "azureAutoApprove": true
  },
  "packageRules": [
    {
      "extends": [
        "packages:eslint"
      ],
      "matchUpdateTypes": [
        "digest",
        "patch",
        "minor"
      ],
      "groupName": "eslint packages",
      "automerge": true,
      "platformAutomerge": true,
      "azureAutoApprove": true
    },
    {
      "extends": [
        "packages:eslint"
      ],
      "matchUpdateTypes": [
        "major"
      ],
      "groupName": "eslint packages (major)",
      "enabled": false
    },
    {
      "matchUpdateTypes": [
        "minor",
        "patch",
        "pin",
        "digest"
      ],
      "automerge": true,
      "platformAutomerge": true,
      "azureAutoApprove": true
    }
  ]
}
{% endraw %}
```

Now if you run Renovate, it will raise PR with requester added and PR is approved by the requester. 

![2022-10-27-azure-auto-approve.png](/assets/images/posts/2022-10-27-azure-auto-approve.png)

### Setup pipeline script

In the CI pipeline, mentioned above, we will add a step with bash script to get the requester's `vote` status from the PR and then update all the reviewers' `vote` status.

It looks like Azure API doesn't allow to update all reviewers with one request. You can only update yours but not others (even with the admin privilege). So we will get the default reviewers' reviewer IDs and use their PAT (Personal Access Token) to automatically update each reviewer's `vote`. 

Go to your pipeline setting and add variables. For the PAT variables, make sure to check "Keep this value secret" to secure the PAT. You can get each reviewer's id by using <a href="https://learn.microsoft.com/en-us/rest/api/azure/devops/git/pull-request-reviewers/list?view=azure-devops-rest-6.0&tabs=HTTP" target="_blank">Azure Pull Request Reviewers - List API</a>

<img src="/assets/images/posts/2022-10-27-pipeline-variables.png" width=500 />

In the CI pipeline's `yml` file, we can access those variables and call the Azure API for each reviewer. 

```yml
{% raw %}
# CI pipeline's yml file

...

variables:
  - name: isRenovatePR
    value: $[startsWith(variables['system.pullRequest.sourceBranch'], 'refs/heads/renovate')]
  - name: SERVICE_ACCOUNT_REVIEWER_ID
    value: a12b6b5d-a224-4183-8a98-517e0a7279d1

stages:
  - stage: CI
    jobs:
      - job: BuildTest
        displayName: Build and Test
        pool:
          vmImage: ubuntu-20.04
        steps:
          - checkout: self
          - template: templates/install-node.yml
          - template: templates/npm-authenticate.yml
          - template: templates/install-project-dependencies.yml
          - template: templates/validate-pr-title.yml
          - template: templates/lint-test-build.yml
          - template: templates/auto-approve-pr.yml
            parameters:
              reviewers:
                - id: $(REVIEWER_1_ID)
                  pat: $(REVIEWER_1_PAT)
                - id: $(REVIEWER_2_ID)
                  pat: $(REVIEWER_2_PAT)
                - id: $(REVIEWER_3_ID)
                  pat: $(REVIEWER_3_PAT)

...
{% endraw %}
```

```yml
{% raw %}
# templates/auto-approve-pr.yml

parameters:
  - name: reviewers
    type: object
    default:
      - reviewers:
          - id: ''
            pat: ''

steps:
  - ${{ each reviewer in parameters.reviewers }}:
      - script: |
          set -e
          VOTE=$(curl -u azdo:$(System.AccessToken) \
            $(System.CollectionUri)_apis/git/repositories/$(Build.Repository.ID)/pullRequests/$(System.PullRequest.PullRequestId)/reviewers/$(SERVICE_ACCOUNT_REVIEWER_ID)?api-version=6.0 \
            | jq '.vote')
          REVIEWER_PAT=${{ reviewer.pat }}
          BASE64_PAT=$(printf "%s"":$REVIEWER_PAT" | base64)
          REVIEWER_ID=${{ reviewer.id }}
          REVIEWER_BODY=$(jq --null-input '{"vote": '"$VOTE"'}')
          echo $REVIEWER_BODY

          AUTO_APPROVAL=$(curl \
            -X PUT $(System.CollectionUri)_apis/git/repositories/$(Build.Repository.ID)/pullRequests/$(System.PullRequest.PullRequestId)/reviewers/"$REVIEWER_ID"?api-version=6.0 \
            -H 'Accept: application/json' \
            -H "Authorization: Basic $BASE64_PAT" \
            -H 'Content-Type: application/json' \
            -d "$REVIEWER_BODY")
          echo $AUTO_APPROVAL
        displayName: Auto approve PR for ${{ reviewer.id }}
        continueOnError: true
        condition: and(succeeded(), eq(variables.isRenovatePR, true), eq(variables['Build.Reason'], 'PullRequest'))
{% endraw %}
```

Now if you run Renovate, it will raise PR, run the CI pipeline and when everything is passing, it will automatically approve for all reviewers.

![2022-10-27-azure-auto-approve-with-script.png](/assets/images/posts/2022-10-27-azure-auto-approve-with-script.png)

## Conclusion

Now you only need to deal with PRs with major version upgrades and any non-major upgrades breaking the CI pipeline. 🙌
