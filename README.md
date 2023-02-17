# Build and deploy pipelines

Building a deploying multiple versions of software components changed dramatically in the last years.
Where we sometimes could have as few as one deployment per quarter, it's now frequent to see several deployments a day.

More frequent deployments means more frequent testing, from the development team as well as from the final users.
This fast feedback loop allows to develop new functionalities and fix bugs much faster.
But more frequent deployments also implies a more complex and dynamic process.

# An example of implementation of Continuous Integration and Continuous Deployment (CICD) pipelines

In this article, we will discuss a concrete case, developed for one of our main clients.

## General principles of the CICD pipelines

### The development stages

We have 4 different stages on 4 corresponding environments:
- development: where every single new software build will be deployed and can be tested independently 
- integration: where software components are put and tested together
- staging: where software components are deployed in a prod-like environment
- prod: the stable environment that can be accessed by the final users.

TODO include image distinct stages

The application is deployed successively from the development environment to the production environment
following a fixed process.

![Application deployed from dev to prod](./C2C-Blog-article_Deployment-from-dev-to-prod.png)

### The application development lifecycle

The way to build and deploy a software can be described in a pretty generic way since the principles are often
very similar whatever the language and technologies.

![generic build and deployment process](./C2C-Blog-article_source_code_to_deployed_app_principle.png)

This generic process is then implemented differently according to the language and the tools used.

![Example of a Java/Maven Build with Helm/Helmfile deployment in Kubernetes](./C2C-Blog-article_source_code_to_deployed_app_example.png)



## Build docker images with Kaniko

## Detect vulnerabilities in the source code, SAST

## Detect vulnerabilities in the Docker containers

## Manage Docker build dependencies

## Improve Java builds with build caches

# What could be imroved

Test environments dynamically created for each test, each branch, each feature, each bug fix
