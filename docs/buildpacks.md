# Packaging OCI Images with Buildpacks

This branch contains a modification to the current `.devpack-for-spring/plugin-configuration.yaml` that integrates plugin support for packaging OCI container images with buildpacks. In particular it enables the following buildpack integrations:

- The default Paketo builder shipped with the Spring Boot build plugin
- Tanzu Full: tanzu-build.packages.broadcom.com/tanzu-java-buildpack/java
- Tanzu Lite: tanzu-build.packages.broadcom.com/tanzu-java-buildpack/java-lite

## Prerequisites

The `devpack-for-spring` command line tool can be used to set up the development environment for Docker.

```shell
sudo snap install --classic devpack-for-spring
devpack-for-spring setup --add docker
```

> **Note:** Docker can also be installed as noted in [their installation](https://docs.docker.com/engine/install/ubuntu/) steps or manually with the [Docker snap](https://github.com/canonical/docker-snap).

To access the Tanzu image builders, authentication is required to pull then from their associated registry:

```shell
docker login tanzu-build.packages.broadcom.com
```

## Getting Started

After installing Devpack for Spring and Docker, we can either initialize a new Spring Boot application or augment an existing project. For demonstration purposes, we will clone the [PetClinic Sample Application](https://github.com/spring-projects/spring-petclinic) from Spring.

```shell
# Cloning an existing project
git clone https://github.com/rroessler/spring-petclinic.git
cd spring-petclinic
git checkout tanzu-buildpacks

# Starting a new Spring project
devpack-for-spring boot start
```

> **Note:** We use a modified PetClinic fork as Paketo builders require OpenJDK 25 and above, whilst PetClinic sets this to Java 17 for Gradle.

To then ensure the development environment is correct we will need to run our setup command again for the PetClinic project:

```shell
devpack-for-spring setup --add gradle openjdk-17-jdk
```

We can then configure plugin support for buildpacks by copying this branches `.devpack-for-spring/plugin-configuration.yaml` file. In future this would be simplified by instead running:

```shell
devpack-for-spring init-plugin
```

Alternatively for demonstration purposes, the following snippet can be placed in the `.devpack-for-spring/plugin-configuration.yaml` file:

```yaml
buildpack:
  gradle:
    id: org.springframework.boot
    version: 4.1.1
    default-task: paketo-base
    repository: gradlePluginPortal()
    description: |
      Plugin for OCI image generation using Cloud Native Buildpacks
    tasks:
      paketo-base: bootBuildImage
      tanzu-lite:
        - bootBuildImage
        - --builder=tanzu-build.packages.broadcom.com/tanzu-java-buildpack/java-lite
      tanzu-full:
        - bootBuildImage
        - --builder=tanzu-build.packages.broadcom.com/tanzu-java-buildpack/java
  maven:
    id: org.springframework.boot
    version: 4.1.1
    default-task: paketo-base
    description: |
      Plugin for OCI image generation using Cloud Native Buildpacks
    tasks:
      paketo-base: :build-image
      tanzu-lite:
        - :build-image
        - --builder=tanzu-build.packages.broadcom.com/tanzu-java-buildpack/java-lite
      tanzu-full:
        - :build-image
        - --builder=tanzu-build.packages.broadcom.com/tanzu-java-buildpack/java
```

## Running the Buildpack Plugin

From here running the buildpack plugin can be done with the following commands:

```shell
devpack-for-spring run buildpack                # Pack with default Paketo builder
devpack-for-spring run buildpack tanzu-full     # Pack with commercial Tanzu builder
```

> **Note:** If this command errors with Gradle not being able to find a Java installation on your machine, this means that the `JAVA_HOME` environment variable is missing. This can be faked by applying an environment to the command: `JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64 devpack-for-spring run buildpack`

Assuming Docker is installed and running correctly, this will output a packaged container image of your application with "NAME:VERSION" as it's name.

```shell
# Helper to see all images available
docker image ls

# Helper to show just the packaged project
docker image ls --filter "reference=NAME"
```

The project image can then be ran using the standard `docker run` command:

```shell
docker run -p 8080:8080 docker.io/library/NAME:VERSION
```

## Pruning Docker

As a helpful utility, the following commands can be used when testing to prune _ALL_ Docker related artifacts:

```shell
docker rmi $(docker images -a -q)       # Remove built images
docker system prune -a --volumes -f     # Remove everything
```

> **Note:** These should be used with caution if you have other Docker artifacts on your system.
