---
layout: page
title: Uploading maven packages
permalink: /docs/uploading-maven-packages/
---

The gradle build system can pull in dependencies on its own. To make it as easy as possible to work with any IRMA package, we add our own libraries to a maven repository as well. To make it as easy as possible for developers to upload packages, the maven repository is included the GitHub pages ([credentials.github.io](https://github.com/credentials/credentials.github.io/)) under `repos/maven2`. This short document describes how to upload new packages.

Furthermore, when you're working from scratch, it might be useful to consider the following dependency graph:
![IRMA dependency graph](/images/dependencies.svg)
An arrow encodes the "is a dependency of" relationship.

## Short guide

See below for a more verbose guide.

 1. Set the `mavenRepositoryIRMA` Gradle property to `repos/maven2` in your checkout of [credentials.github.io](https://github.com/credentials/credentials.github.io).
 2. Set the correct verion in `build.gradle` of the project you're uploading.
 3. Run `gradle uploadArchives` to build the relevant files
 4. Commit these files in your `credentials.github.io` repository.
 5. Make sure you push your commit to publish the required files.

## Long guide

All relevant Gradle build files (with the notable and deliberate exception of that for [idemix_library](https://github.com/credentials/idemix_library)) are configured to upload a new version of that library to a maven repository stored in the directory pointed to by `mavenRepositoryIRMA`. The easiest way to set this property is to set it globally. To do so, add

    mavenRepositoryIRMA=/path/to/checkout/of/credentials.github.io/repos/maven2

to `~/.gradle/gradle.properties`. Make sure that the path points to the directory `repos/maven2` inside your checkout of `credentials.github.io`.

Next, check that you've set the correct version in the Gradle build file. Then run

    gradle uploadArchives

Gradle will compile all the relevant files, and put them in `repos/maven2` of your GitHub pages checkout. Look them over to see if they match your expectations. Then add the files, and commit them. Finally, push your commits to publish the files.
