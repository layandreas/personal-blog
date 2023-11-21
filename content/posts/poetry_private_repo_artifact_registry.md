+++
author = "Andreas  Lay"
title = "Setting Up Poetry to Access GCP Artifact Registry"
date = "2023-11-20"
description = "How to use Poetry and GCP Artifact Registry"
tags = ["python", "poetry", "gcp"]
categories = ["Python"]
ShowToc = true
TocOpen = true
+++

## Introduction

In a corporate setting sooner or later you will want to host your in-house Python packages on a private artifact store. We are using [Poetry](https://python-poetry.org/) for our package management and using Google Cloud as our cloud provider, therefore [Artifact Registry](https://cloud.google.com/artifact-registry) is our store of choice. However the combination of `Poetry` with `Artifact Registry` is not well documented, so I hope this post helps. [Creating a private package repository on GCP](https://cloud.google.com/artifact-registry/docs/python/store-python#create) itself is straightforward and I assume you already have created one. The focus of this post is authentication.

## Publishing a Private Package
This is how you can publish a package to your private repository:

```bash
# Set the private repo as a global Poetry config
poetry config repositories.private-repo https://<REGION>-python.pkg.dev/<PROJECT_ID>/<REPO_NAME>/

# Let's create a new Python package using Poetry
poetry new my_private_package
cd my_private_package

# We will use the http basic auth but our "password" is a GCP access token
# and our "user" is "oauth2accesstoken"
# See also here: https://cloud.google.com/artifact-registry/docs/docker/pushing-and-pulling#token
export POETRY_HTTP_BASIC_PYPI_USERNAME=oauth2accesstoken
export POETRY_HTTP_BASIC_PYPI_PASSWORD=$(gcloud auth print-access-token)

# Publish this package to the private repo
poetry publish --build --repository private-repo
```

As you can see this is really simple, however it's unintuitive and not well-documented that artifact registry uses basic auth headers for what is actually a bearer auth (see [authentication schemes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication#authentication_schemes)). Note that you will need to install the [gcloud CLI](https://cloud.google.com/sdk/docs/install) to fetch an access token.


## Adding a Private Package as Dependency

Now that we have published a package we can add it as dependency to other projects and/or packages. Just create a new project using Poetry and add your private repo as a supplemental source:

```bash
# Let's create a new Poetry project
poetry new myproject
cd myproject

# We add pypi as our primary source and our private repo as a supplemental source
poetry source add --priority=primary PyPI
poetry source add --priority=supplemental private-repo https://<REGION>-python.pkg.dev/<PROJECT_ID>/<REPO_NAME>/simple/

# Authenticate
export POETRY_HTTP_BASIC_PYPI_USERNAME=oauth2accesstoken
export POETRY_HTTP_BASIC_PYPI_PASSWORD=$(gcloud auth print-access-token)

poetry add --source private-repo my_private_package
```
