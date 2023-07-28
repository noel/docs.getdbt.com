---
title: "Advanced: Create a release train with additional environments"
slug: multiple-environments
description: Large and complex enterprises sometimes require additional layers of validation before deployment. Learn how to add these checks with dbt Cloud.
---

:::caution Are you sure you need this?
This approach creates additional manual steps in the deployment process as well as a greater maintenance burden.

As such, it may slow down the time it takes to get new features into production.

The team at Sunrun maintained a SOX-compliant deployment in dbt while reducing the number of environments. Check out [their Coalesce presentation](https://www.youtube.com/watch?v=vmBAO2XN-fM) to learn more.
:::

## Prerequisites

This section assumes you already have the **Development**, **Staging** and **Production** environments described in [the Quickstart](/guides/orchestration/set-up-ci/in-15-minutes).

In this section, we will add a new **Release** environment. New features will branch off from and be merged back into the associated `release` branch, and a member of your team (the "Release Manager") will create a PR against `main` to be validated in the Staging environment before going live.

## Create a `release` branch in your git repo

As noted above, this branch will outlive any individual feature, and will be the base of all feature development for a period of time. Your team might choose to create a new branch for each sprint (`release/sprint-01`, `release/sprint-02`, etc), tie it to a version of your data product (`release/1.0`, `release/1.1`), or just have a single `release` branch which remains active indefinitely.

## Update your Development environment to use the `release` branch

See [Custom branch behavior](/docs/dbt-cloud-environments#custom-branch-behavior). Setting `release` as your custom branch ensures that the IDE creates new branches and PRs with the correct target, instead of using `main`.

## Create a new Release environment

See [Create a new environment](/docs/dbt-cloud-environments#create-a-deployment-environment). The environment should be called **Release**. Just like your existing Production and Staging environments, it will be a Deployment-type environment.

Set its branch to `release` as well.

## Create a new job

Use the **Continuous Integration Job** template, and call the job **Release CI Check**.

In the Execution Settings, your command will be preset to `dbt build --select state:modified+`. Let's break this down:

- [`dbt build`](/reference/commands/build) runs all nodes (seeds, models, snapshots, tests) at once in DAG order. If something fails, nodes that depend on it will be skipped.
- The [`state:modified+` selector](/reference/node-selection/methods#the-state-method) means that only modified nodes and their children will be run ("Slim CI"). In addition to [not wasting time](https://discourse.getdbt.com/t/how-we-sped-up-our-ci-runs-by-10x-using-slim-ci/2603) building and testing nodes that weren't changed in the first place, this significantly reduces compute costs.

To be able to find modified nodes, dbt needs to have something to compare against. Normally, we use the Production environment as the source of truth, but in this case there will be new code merged into `release` long before it hits the `main` branch and Production environment. Because of this, we'll want to defer the Release environment to itself.

### Optional: also add a compile-only job

Even when deferring to its own environment, dbt Cloud uses the last successful run of any job in that environment as its [comparison state](/reference/node-selection/syntax#about-node-selection). If you have a lot of PRs in flight, the comparison state could switch around regularly.

Adding a regularly-scheduled job inside of the Release environment whose only command is `dbt compile` can regenerate a more stable manifest for comparison purposes.

## Some heading here

When the Release Manager is ready to cut a new release, they will open a PR from `release` into `main` from their git provider, at which point the existing CI Check in the Staging environment will trigger and run.

To test your new flow, create a new branch in the dbt Cloud IDE then add a new file or modify an existing one. Commit it, then create a new Pull Request (not a draft). Within a few seconds, you’ll see a new check appear in your git provider.