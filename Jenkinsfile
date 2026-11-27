#!groovy
// MAMA build: only the apiserver container image, pushed to our registry.
// Pipeline source: https://github.com/the-mama-ai/jenkins-build-shared (vars/mamaBuildDockerImage.groovy)

// Upstream's image is built from Maven output (see `make build-image`). The agents have no JDK,
// so Maven runs in a container; the named volume keeps ~/.m2 (repository + build cache) between builds.
def MAVEN_IMAGE = 'docker.io/library/maven:3.9-eclipse-temurin-25'

def pomVersion = {
    // the root pom has no <parent>, so its first <version> is the project's own
    sh(returnStdout: true, script: "grep -m1 -oP '(?<=<version>)[^<]+' pom.xml").trim()
}

def buildJava = {
    env.APP_VERSION = pomVersion()
    sh """#!/bin/bash -eux
        podman run --rm --security-opt label=disable \
            -v "\$PWD:/work" -v dependency-track-m2:/root/.m2 -w /work \
            ${MAVEN_IMAGE} \
            mvn -B -Pquick -pl apiserver -am package

        # The pipeline always builds with the repo root as context, while upstream's Dockerfile
        # and .dockerignore expect apiserver/ — stage what they COPY at the root instead of forking them.
        rm -rf target src
        cp -r apiserver/target target
        mkdir -p src/main && cp -r apiserver/src/main/docker src/main/docker
    """
}

mamaBuildDockerImage(
    // SNAPSHOT is dropped, the pipeline appends its own pre-release suffix to branch builds
    versionProvider: { pomVersion().replaceFirst(/-SNAPSHOT$/, '') },
    dockerfile: 'apiserver/src/main/docker/Dockerfile',
    dockerArgs: ['--ignorefile', 'apiserver/.dockerignore'],
    // single-quoted: expanded by the shell at build time
    dockerBuildArgs: ['APP_VERSION=$APP_VERSION', 'COMMIT_SHA=$GIT_COMMIT'],
    customStages: [preBuild: buildJava],
    publishBranches: ['mama-extra'],
)
