# Build and deploy pipelines

Building and deploying multiple versions of software components changed dramatically in the last years.
Where we sometimes could have as few as one deployment per quarter, it's now frequent to see several deployments a day.

More frequent deployments means more frequent testing, from the development team as well as from the final users.
This fast feedback loop allows to develop new functionalities and fix bugs much faster.
But more frequent deployments also implies a more complex and dynamic process.

# An example of implementation of Continuous Integration and Continuous Deployment (CICD) pipelines

In this article, we will discuss a concrete case, developed for one of our main clients.

1. [General principles of the CICD pipelines](#General-principles-of-the-CICD-pipelines)
2. [Detect vulnerabilities in the source code, SAST and DAST](#detect-vulnerabilities-in-the-source-code-sast-and-dast)
3. [Build docker images with Kaniko](#Build-Docker-images-with-Kaniko)
4. [Detect vulnerabilities in the Docker containers](#Detect-vulnerabilities-in-the-Docker-containers)
5. [Manage Docker build dependencies](#Manage-Docker-build-dependencies)
6. [Improve Maven Java builds with build caches](#Improve-Maven-Java-builds-with-build-caches)

## General principles of the CICD pipelines

### The development stages

We have 4 different stages on 4 corresponding environments:
- development: where every single new software build will be deployed and can be tested independently 
- integration: where software components are put and tested together
- staging: where software components are deployed in a prod-like environment
- prod: the stable environment that can be accessed by the final users.

The application is deployed successively from the development environment to the production environment
following a fixed process.

![Application deployed from dev to prod](./C2C-Blog-article_Deployment-from-dev-to-prod.png)

### The application development lifecycle

The way to build and deploy a software can be described in a pretty generic way since the principles are often
very similar whatever the language and technologies.

![generic build and deployment process](./C2C-Blog-article_source_code_to_deployed_app_principle.png)

This generic process is then implemented differently according to the language and the tools used.

![Example of a Java/Maven Build with Helm/Helmfile deployment in Kubernetes](./C2C-Blog-article_source_code_to_deployed_app_example.png)

## Detect vulnerabilities in the source code, SAST and DAST

A lot of plugin and tools allow developers to search for potential issues before they occur.
We speak mainly about Static Application Security Testing (SAST) and Dynamic Application Security Testing (DAST).
The Static Security Testing consist of analysing the source code for known bad patterns and the dependencies for
dependencies known to contain vulnerabilities.

One of the most know static analysis application is ![Sonarqube](https://www.sonarsource.com/products/sonarqube/). This is a standalone application that can be linked with an
existing development environment. ![Gitlab](https://about.gitlab.com), an end-to-end software development platform, integrates natively security testing
plugins which we just need to activate to get security results aggregated into the dedicated dashboard.

Once the Gitlab plugins are activated, the vulnerability report can be reviewed directly via Gitlab's interface.

![Gitlab Security Dashboard](./Gitlab_sec_dashboard.png)

Those results can eventually be pulled via the Gitlab API and included into your own customized pipelines.

![Gitlab Security Pipeline](./Gitlab_sec_pipeline.png)

Doing so, you can define your own ![quality gates](https://docs.sonarqube.org/latest/user-guide/quality-gates/)

## Build docker images with Kaniko

Building Docker images became a very common task in modern development environments.
Since the development environments are often hosted on Docker environments, we regularly face Docker in Docker issues:
building a Docker container in a Docker container requires to bypass important security features and is therefore unsafe.

Moreover, Docker container runtime has been ![Deprecated by Kubernetes](https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/) and we need a solution to build such containers on top of container engines like ![Podman](https://podman.io/) or ![CRI-O](https://cri-o.io/).

Luckily, Google released a tool named ![Kaniko](https://github.com/GoogleContainerTools/kaniko) that we can easily ![use from Gitlab](https://docs.gitlab.com/ee/ci/docker/using_kaniko.html)
To simplify the whole process and allow developers to specify their build parameters in a convenient way, we developed a
custom process that analyzes the `docker-compose.yaml` to extract the build information : image name, image tag, build variables, etc.
Once the required information are acquired, we create a ![dynamic child pipeline](https://docs.gitlab.com/ee/ci/pipelines/downstream_pipelines.html#dynamic-child-pipelines) which is gonna take care of the build of the image.

We decided to use the format of `docker-compose` as a DevOps interface because :
- it is widely used as a standard among developers that are running the containers on their workstations. Therefore the pipeline should support it as well.
- it allows the developer to pass build parameters into the build pipeline.
- it allows to build several images from a single Gitlab project.

Dynamic Child Pipelines are a version of child pipelines wherein the child pipeline can be generated within a job or a set of jobs in the parent pipeline. The parent pipeline must put the generated CI configuration in an artifact, and then the trigger job refers to that artifact to tell the CI system what to run.

In a nutshell, our pipeline reads the `docker-compose.yml` file to get build information which will be passed to Kaniko in order to build the Dockerfile present in your project.

## Detect vulnerabilities in the Docker containers

Once built, our images are deployed to a ![quay](https://quay.io) registry. This tool already includes the feature of analysing the container layers against know vulnerabilities. In our pipelines, we implemented a python script which requests the results of this analysis, what allows us to define some quality-gates in the deployment process. We can for example prevent any image with know severe vulnerabilities to be deployed in any of our environments.

## Manage Docker build dependencies

In most of our projects, we create Docker containers, either based on images from ![DockerHub](https://hub.docker.com/) or internally built images. To ensure that our downstream images include all the latest security patches, we need to build a new version of the downstream image every time an upstream parent image is updated.

To achieve this, we use a GitOps approach : GitOps is a software development methodology that uses Git as the single source of truth for declarative infrastructure and application configuration. It involves using Git to store and manage the desired state of infrastructure and applications, and using automated processes to continuously synchronize the actual state with the desired state. This allows for efficient and consistent management of infrastructure and applications across different environments, and enables faster and more reliable delivery of software.

Following this GitOps approach, we create a file named `.parent-images/<image-name>-<commit>` that links the upstream image to the code repository. This allows us to trigger a build automatically every time a new commit is made to the repository, ensuring that our downstream images are always up to date.

To manage the dependencies between our projects and containers, we developed a dedicated Python program. The program analyzes the Dockerfile of each project in pre-specified groups, building two dependency trees: one mapping GitLab projects to the container images they create, and another mapping container images to their parent images. By merging these two trees, we can determine which GitLab projects depend on which other projects and therefore, which downstream projects need to be rebuilt after a successful build of a specific project.

![Dependencies tree](./Build-dependencies-tree.png)

The Python program then adds or updates the "projects-to-trigger.txt" file, which contains the list of downstream projects to be triggered after a build in every affected project.
After a successful build, the pipeline reads the "projects-to-trigger.txt" file and loops over its entries. This creates or updates the `.parent-images/<image-name>-<commit>` file in every downstream projects listed in the file, which triggers a build on those projects

By using the GitOps methodology, we can efficiently ensure that all downstream images are updated, secure, and reflecting the latest changes made to their parent images.

## Improve Maven Java builds with build caches

Maven is a wonderful tool allowing you to manage and download automatically all the dependencies you defined for your project.
When the build is running on Gitlab, it can run on different runners each time and therefore, might download hundreds of megabytes
every single time the build runs.
To mitigate this problem, we can make use of the cache functionality. To do so, we define the `.m2/repository` directory to be cached.
The very first time it will run, all dependencies will be downloaded by Maven. Then, those dependencies will be archived and stored.
During every single next build, this archive will be first expanded and all the previously downloaded dependencies will be made available
to the Maven tool. The later will only have to check and download for updates, if any.
This saves up sometimes more than 30 min per build for projects with a lot of dependencies.

# Conclusion

In this article we described some general principles of a good CICD pipeline and discussed more specific implementation details like different types of security scanning and build optimizations.

There are still a lot of improvements that could be brought to such build and deployment pipelines. We could for example dynamically create test environments for each feature branch or bug fix, containing only those specific changes, in order to test them independently from any other modifications that could have be brought to a shared integration platform.
We could also create dedicated sandbox environments for every single developer, where he would be free to test anything without causing any damage to the shared environments.

A lot of other improvements can be thought of and what you'll be able implement yourself will often depend on the resources and time you have!
