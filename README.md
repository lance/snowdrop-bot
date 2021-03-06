# Table of Contents

   * [Snowdrop Bot](#snowdrop-bot)
      * [Features](#features)
      * [Installation](#installation)
         * [Database](#database)
         * [Github](#github)
         * [Google APIS](#google-apis)
         * [Jira](#jira)
      * [Running](#running)
         * [Default Profile](#default-profile)
            * [Persistence and configuration (defalut)](#persistence-and-configuration-defalut)
         * [Production Profile](#production-profile)
            * [Persistence and configuration (production)](#persistence-and-configuration-production)
      * [Issue tracking](#issue-tracking)
      * [Pull Request tracking](#pull-request-tracking)
      * [Forked repository issue bridging](#forked-repository-issue-bridging)
      * [Google Docs Report Generation](#google-docs-report-generation)
      * [Kubernetes / Openshift deployment](#kubernetes--openshift-deployment)


# Snowdrop Bot
A bot for automating snowdrop related tasks

## Features

- [Issues tracking](#issue-tracking) across multiple organizations and repositories
- [Pull request tracking](#pull-request-tracking) across multiple organization and repositories
- [Forked repository issue bridging](#forked-repository-issue-bridging)
- [Google Docs report generation](#google-docs-report-generation)

## Installation

To prevent adding personal information to github, create an alternate resources folder at `src/main/local-resources` and copy the `src/main/resources/application.properties`
file into it. All changes should be made to this file. This folder is excluded in `.gitignore`.

### Database

For the persistence needs of this application `h2` has been used.
The database is configured to store files under `~/.snowdrop-bot/db`.
So you will either need to create the folder and ensure that you have write
access, or reconfigure the application to store files somewhere else.

You can specify where h2 will store files by editing: `quarkus.datasource.url`.

### Github

The application will need to access github. So you are going to need an access
token.

The token can be configured either via `github.token` property or `GITHUB_TOKEN`
environment variable.

> NOTE: The `github.token` property is configured to have it's value filled with the [Maven Rsource plugin Filter capabilities](https://maven.apache.org/plugins/maven-resources-plugin/examples/filter.html).
> It can be easilly filled by adding -Dgithub.token=<the token> to the maven compilation statement.

### Google APIS

To upload generate reports to Google Documents, you will need to enable the
Google Documents API. Once enabled, you will need to store the provided
`credentials.json` file under `/.snowdrop-bot/google/credentials.json`.
Again the location is configurable via `google.docs.credentials.file` property.

### Jira

The application will need to access jira nd for that it will need the credentials.

This information can be provided either by the application.properties file or environment variables.

`application.properties`:

```properties
jira.user=my-jira-user
jira.password=my-jira-password
```

Environment variables:

```bash
$ export JIRA_USERNAME=my-jira-user
$ export JIRA_PASSWORD=my-jira-password
```

## Running

To run the bot you can just build:

    ./mvnw clean package

If using resource filtering use build it this way instead:

    ./mvnw -Dgithub.token=<the token> clean package

and then run with:

    java -jar target/snowdrop-bot-0.1-SNAPSHOT-runner.jar

*NOTE:* The `github.token` is required, so either set it in
`application.properites` or create an environment variable for it.

### Default Profile
To avoid unnecessary traffic on the github API (which is subject of rate
limiting) the default profile has the core features (e.g. bridge and reporting disabled).

You can enable them selectively through the UI, or use a different profile (e.g.
production).

#### Persistence and configuration (defalut)
The default profile uses /data/snowdrop-bot as the root for configuration and database files.


### Production Profile
The production profile has all features enabled by default.
Additionally, everything related to file persistence or file configuration is
configured under `/data/snowdrop-bot` which which is a volume friendlier path
(e.g. when running on Kubernetes).

#### Persistence and configuration (production)
The production profile uses /data/snowdrop-bot as the root for configuration and database files.

## Associate list

The associate table can be maintained with the following REST services:

```bash
$ curl -X PUT -d associate=<github-id> -d alias=<bot-alias> -d source=[GITHUB,JIRA,...] -d name="<Associante Name>" localhost:8080/associate/create
```

A list of the associates can be obtained using the list service.

```bash
$ curl -X GET localhost:8080/associate/list
```

## Issue tracking

Provides a unified view of all `active` issues across multiple repositories.
The criteria for selecting issues are the following:

- exist in repositories of configured organizations (`github.reporting.organizations`)
- assigned to configured users (`github.users`)
- the assigned user has forked the repository
- were open within the configured time frame (the week bounded by `github.reporting.day-of-week` & `github.reporting.hours`)

Those issues, can be exported in csv, excel or pdf.

**NOTE:** At the moment we don't track the activity of the issue in any way. In
other words, we do not filter out `stale` issues.

![issues screen](./img/issues.png "Issues Screen")

## Pull Request tracking

Provides a unified view of all `active` pull requests across multiple repositories.
The criteria for selecting issues are the following:

- exist in repositories of configured organizations (`github.reporting.organizations`)
- assigned to configured users (`github.users`)
- the assigned user has forked the repository
- were open within the configured time frame (the week bounded by `github.reporting.day-of-week` & `github.reporting.hours`)

![pull request screen](./img/pull-requests.png "Pull Requests Screen")

Those pull requests, can be exported in csv, excel or pdf.

**NOTE:** At the moment we don't track the activity of the pull requests in any way. In
other words, we do not filter out `stale` pull requests.

## Forked repository issue bridging

When working across multiple different organizations owned by different teams
with variable access levels, its important to be able to have an aggregated view
of the pending issues. This is important for things like:

- tracking
- scheduling
- reporting

Lack of permissions in 3rd party repositories make it difficult to manage and
aggregate. In other cases its not allowed to track repositories across
organizations.

Whatever, the reason forking the repository and using a bridge to `clone` issues
of interest is a possible solution.

This bot provides this feature and allows you to configure the following
options:

- source repositories (`github.bridge.source-repos`)
- target organization (`github.bridge.target-organization`)
- terminal label (the label to use to mark an issue as closed `github.bridge.terminal-label`)


## Google Docs Report Generation

It's possible to combine information found in `issues` and `pull requests` and
present them in `Google Docs`.

This can be done by selecting the `Reporting` tab and clicking the `generate` button.

This requires the setup mentioned in [Google APIS](#google-apis).
The id of the target document can be specified using `google.docs.report.document-id`.


## Kubernetes / Openshift deployment

During compilation resources for Kubernetes and Openshift are generated.

Before these resources can be applied, some preparation is required.

1. Create a PV with name `snowdrop-db-claim`
2. Create a secret with the github token.

``` sh
kubectl create secret generic dev-db-secret --from-literal=GITHUB_TOKEN=<your github token here>
```

Since the snowdrop team is using a shared `password-store` for handling
credentials, the following would work:

``` sh
export GITHUB_TOKEN=`pass show snowdrop/github.com/snowdrop-bot/token`
kubectl create secret generic dev-db-secret --from-literal=GITHUB_TOKEN=$GITHUB_TOKEN
```

`
3. Create a secret with the googledocs `credentials.json` file

``` sh
kubectl create secret generic snowdrop-googledocs --from-file=credentials.json=/path/to/credentials.json
```

4. Build the container and apply the manifests.

``` sh
mvn clean package -Dquarkus.kubernetes.deploy=true
```

The project is configured to use `Openshift` out of the box.
To use `Kubernetes` you may need to set
`quarkus.kubernetes.deployment-target=kuberentes` to `application.properties`
and also add container-image-docker or container-image-jib dependencies to your pom.
