# Gradle: Publish a Multi-Module Java Project to Maven Central

While you have consumed many dependencies from Maven Central (e.g., `com.google.guava:guava`),
what if you want to publish to Maven Central? While you could figure it out from various sources of
documentation&mdash;and the Gradle slack channel&mdash;an all-in-one guide would still be immensely helpful.
I wish I had such a guide when I went through this process, so I've written this guide for you.

This guide is split into multiple steps. A key feature of this guide is that  it will also show you how to verify
most of these steps&mdash;since mistakes are often inevitable. As a result, by the time you reach the end&mdash;when
you are automatically staging Maven Central releases via a GitHub workflow&mdash;hopefully everything will Just Work™.

## Soft Assumptions

The guide will still mostly work if some of these assumptions do not hold, but a few steps might be different.
In many cases, I've provided some brief notes on what those differences would be.

Soft Assumptions:

- You are publishing a multi-module Java project using Gradle.
- You may have one or more modules that do not get published (e.g., an example module).
- The project that you are publishing is a personal project that is hosted on GitHub.
- You are publishing artifacts under a group that starts with `io.github.[username]`.
- You will first create new release in GitHub, and then stage a corresponding release in Maven Central
  using a GitHub workflow.
    - The GitHub release process is separate from the Maven Central release process.

*Note: To create this guide, I used Gradle 8.5 with the Kotlin DSL.*

## High-Level Requirements

Here are the high-level requirements to publish to Maven Central:

- You must prove that you own the group that you are publishing under.
    - E.g., Google owns the `com.google.guava` and `com.google.dagger` groups. 
- You must provide sources and Javadoc JARs.
- You must provide a Maven POM with "sufficient" metadata.
- You must provide checksums for the published files.
- You must sign the published files using GPG/PGP.

## Steps

### Step 1: Create a Maven Central Account

[https://issues.sonatype.org/secure/Signup!default.jspa](https://issues.sonatype.org/secure/Signup!default.jspa)

Notes:

- This account is not the same as an account for the [Publisher Early Access][central-early-access] portal;
  an account for that portal will not work here.
- In this guide, we'll use the terms Maven Central and OSSRH interchangeably.
   - (There are differences between the two, but they are not as relevant for this guide.)
- In case they go out-of-date, the links for steps 1 and 2 originally came
  from the Maven Central [Getting started][central-getting-started] page.

### Step 2: Prove Ownership of the Publishing Group

If you are publishing under a group that starts with `io.github.[username]`, this process is automated.
(If not, see the Maven Central documentation for [coordinates][central-coordinates].)

[https://issues.sonatype.org/secure/CreateIssue.jspa?pid=10134&issuetype=21](https://issues.sonatype.org/secure/CreateIssue.jspa?pid=10134&issuetype=21)

After you create the ticket, create a public, empty GitHub repository whose name is the ticket ID
(e.g., `OSSRH-97901`).

To verify this step, "Bot Central-OSSRH" should automatically approve your ticket with a small delay.

**Publishing Note**

You will be granted permission to publish not just under the specific group that you requested,
but also under any sub-group of `io.github.[username]`.

### Step 3: Create a Skeleton Convention Plugin

If you are working with multi-module projects, you should be familiar with convention plugins, i.e., `buildSrc`.
Separate from the Java convention plugin, you will create a convention plugin for publishing.
E.g., my convention plugin is `buildSrc/src/main/kotlin/io.github.mikewacker.drift.publish-conventions.gradle.kts`.

Here is the skeleton convention plugin:

```kotlin
plugins {
    java
    `maven-publish`
    signing
}

publishing {
    publications {
        create<MavenPublication>("mavenJava") {
            from(components["java"])
            versionMapping {
                usage("java-api") {
                    fromResolutionOf("runtimeClasspath")
                }
                usage("java-runtime") {
                    fromResolutionResult()
                }
            }
        }
    }
    repositories {
        maven {
            name = "ossrh"
            url = uri("https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/")
            credentials {
                val ossrhUsername: String? by project
                val ossrhToken: String? by project
                username = ossrhUsername
                password = ossrhToken
            }
        }
    }
}
```

(When it comes time to publish a release, the `maven-publish` plugin will automatically add the checksums.)

Let me explain this line:

```kotlin
val ossrhUsername: String? by project
```

This is a Gradle project property. You can set these properties via an environment variable
that starts with `ORG_GRADLE_PROJECT_`. E.g., if present, the `ORG_GRADLE_PROJECT_ossrhUsername` environment variable
will set the value of `ossrhUsername`.

Of course, you will also need to apply this convention plugin to any modules that will be published.

To verify this step, if the convention plugin is set up correctly, and it has been applied to at least one module,
you should see publishing tasks for the `mavenJava` publication (e.g., `generatePomFileForMavenJavaPublication`)
and the OSSRH repository (e.g., `publishAllPublicationsToOssrhRepository`) when you run `./gradlew tasks`.

### Step 4: Add Sources and Javadoc Jars

Add this snippet to the convention plugin:

```kotlin
java {
    withSourcesJar()
    withJavadocJar()
}

tasks.javadoc {
    (options as StandardJavadocDocletOptions).addBooleanOption("Werror", true)
}
```

Setting the `Werror` option for Javadoc is optional. On balance, I've found that these Javadoc rules
(e.g., document every parameter) are beneficial for writing good Javadoc,
even though it occasionally leads to redundant documentation. For additional guidance on writing good Javadoc,
[Stephen Colebourne's guide][joda-javadoc] are [Oracle's guide][oracle-javadoc] are both good resources.

To verify this step, after you build your project, if you look in the `build/libs` folder of any published module,
you should see a `-sources.jar` and a `-javadoc.jar` in addition to the standard `.jar`.

### Step 5: Set the Group, Artifact, and Version

Assuming that your initial release is version `0.1.0`, add these lines to the top level of the convention plugin:

```kotlin
group = "[group]"
version = "0.1.0-SNAPSHOT"
```

In my project, these lines look like this:

```kotlin
group = "io.github.mikewacker.drift"
version = "0.2.0-SNAPSHOT"
```

For each module, the artifact will automatically be set to the name of the module.

To verify this step, after you build your project, if you look in the `build/libs` folder of any published module,
you should see that the version is present in the names of the JARs.

**Publishing Note**

When you create a new release, this is a common convention to follow:

1. Submit a one-line PR to remove `-SNAPSHOT` from the version.
2. Publish the release.
3. After the release is published, submit a one-line PR to upgrade the version to a snapshot for the next release.

### Step 6: Add Required Metadata to the Maven POM

In the `mavenJava` publication of the convention plugin, add a `pom` below the `versionMapping`, using this template:

```kotlin
            pom {
                url = "[project-url]"
                licenses {
                    license {
                        name = "[license-name]"
                        url = "[license-url]"
                    }
                }
                developers {
                    developer {
                        id = "[username]"
                        name = "[name]"
                        email = "[email]"
                        url = "https://github.com/[username]"
                    }
                }
                scm {
                    connection = "scm:git:git://github.com/[username]/[project].git"
                    developerConnection = "scm:git:ssh://github.com:[username]/[project].git"
                    url = "https://github.com/[username]/[project]/tree/main"
                }
            }
```

(If this is an organization's project, you can instead set the `organization` and `organizationUrl` for a `developer`.)

E.g., here is the `pom` block for my project:

```kotlin
            pom {
                url = "https://github.com/mikewacker/drift"
                licenses {
                    license {
                        name = "MIT License"
                        url = "https://opensource.org/license/mit/"
                    }
                }
                developers {
                    developer {
                        id = "mikewacker"
                        name = "Mike Wacker"
                        email = "11431865+mikewacker@users.noreply.github.com"
                        url = "https://github.com/mikewacker"
                    }
                }
                scm {
                    connection = "scm:git:git://github.com/mikewacker/drift.git"
                    developerConnection = "scm:git:ssh://github.com:mikewacker/drift.git"
                    url = "https://github.com/mikewacker/drift/tree/main"
                }
            }
```

Since each module will have a different name and description,
each published module will set those values in `build.gradle.kts`. E.g.:

```kotlin
publishing {
    publications.named<MavenPublication>("mavenJava") {
        pom {
            name = "Drift"
            description = "Lightweight, all-in-one library to rapidly prototype a JSON API."
        }
    }
}
```

To verify this step, run `./gradlew generatePomFileForMavenJavaPublication`. If you look
in the `build/publications/mavenJava` folder of any published module, you can find and inspect the generated POM file.

*(Note: If you have ever created a multi-module project in Maven instead of Gradle, those projects have a root POM
with a list of modules and also any common elements; the POM for each module also references the root POM as its parent.
This process will instead create a standalone POM for each module. However, that is perfectly fine here;
the modules will still be released together.)*

### Step 7: Sign Files

This step is the most complex, so it will be split in multiple parts.

*Note: The commands here were run on Ubuntu 22.04, which has a 2.2.x version of `gpg`.*

**Part (a): Generate a GPG Key Pair**

Run this command:

```
gpg --full-generate-key
```

`gpg` will take you through an interactive process to create the key:

- The default kind of key and keysize are fine.
- For the expiration, either 1 or 2 years is fine.
    - While you don't have to worry about this for now, there is a way to extend the expiration for a key. 
- For the comment, you can use `OSSRH`.
- For the password, you probably do not want to reuse an existing password.

**Part (b): Distribute Your Public Key**

To get the public key, you can run this command:

```
gpg -k
```

The following command will distribute your public key to a keyserver (but use your own public key here):

```
gpg --keyserver keyserver.ubuntu.com --send-keys 2F5D0D4673AC6F0EF05BA763E191FB4B55FA5014
```

To verify this part, visit [https://keyserver.ubuntu.com](https://keyserver.ubuntu.com) and search for your public key.
As an example, here is the URL that I get redirected to after searching for my own public key on the home page:

[https://keyserver.ubuntu.com/pks/lookup?search=2F5D0D4673AC6F0EF05BA763E191FB4B55FA5014&fingerprint=on&op=index](https://keyserver.ubuntu.com/pks/lookup?search=2F5D0D4673AC6F0EF05BA763E191FB4B55FA5014&fingerprint=on&op=index)

**Part (c): Sign Files Using a Regular Key**

We will eventually use an in-memory key, but we should first get signing to work with a regular key
before we switch to an in-memory key.

First, you will need to export your keys. Run this command:

```
gpg --keyring secring.gpg --export-secret-keys > ~/.gnupg/secring.gpg
```

You will also need the key ID, which is the **last eight** characters of the public key&mdash;not the first eight.
E.g., the key ID for `2F5D0D4673AC6F0EF05BA763E191FB4B55FA5014` is `55FA5014`.

Next, enable signing in the convention plugin:

```kotlin
signing {
    sign(publishing.publications["mavenJava"])
}
```

To locally sign files, create properties like this in `~/.gradle/gradle.properties`:

```
signing.keyId=55FA5014
signing.password=[redacted]
signing.secretKeyRingFile=/home/mike/.gnupg/secring.gpg
```

To verify this part, run `./gradlew signMavenJavaPublication`. (Note: This task will not be listed
if you run `./gradlew tasks`.) If you look in the `build/libs` folder of any published module,
you will see a corresponding signature file (`.asc`) for each JAR.

**Part (d): Sign Files Using an In-Memory Key**

Obviously, the `signing.secretKeyRingFile` property will not play well with GitHub workflows.
For a GitHub workflow, we must use an in-memory key instead.

First, export the private key in an ASCII format by running a command like this:

```
gpg --armor --export-secret-key 2F5D0D4673AC6F0EF05BA763E191FB4B55FA5014 > [private-key-file.asc]
```

Next, change the `signing` block in the convention plugin:

```kotlin
signing {
    val signingKey: String? by project
    val signingPassword: String? by project
    useInMemoryPgpKeys(signingKey, signingPassword)
    sign(publishing.publications["mavenJava"])
}
```

To locally sign files, you can create a file that will set the necessary environment variables:

```
export ORG_GRADLE_PROJECT_signingKey=$(cat [private-key-file.asc])
export ORG_GRADLE_PROJECT_signingPassword=[redacted]
```

Then, whenever you need to load these environment variables, you can run the `source` command against this file.

To verify this part, once you load those environment variables, run `./gradlew clean signMavenJavaPublication`
to clean the signature files from part (c) and then re-sign the files.

### Step 8: Publish the First Release Manually

It is recommended that you publish the first release manually. A release involves three steps:

1. Create a staging repository for the release; this is done via `./gradlew publish`.
2. Close the staging repository; validation checks will run at this step.
3. Release the staging repository.
    - Yes, you must "close" a staging repository before you can "release" it; the terminology is a bit awkward there.

First, log in to OSSRH ([https://s01.oss.sonatype.org/](https://s01.oss.sonatype.org/)) with the same username and
password as the JIRA account that you created in step 1. However, you should not use those credentials to publish.
After you log in, you should visit [this page][central-token] to create a user token.
(Note: This is not the same as the tokens that you can create on the JIRA site; those tokens will not work.)

In the file from step 7 that sets the environment variables for signing, you should also set the environment variables 
for `ossrhUsername` and `ossrhToken`, using the user token that you created:

```
export ORG_GRADLE_PROJECT_ossrhUsername=[token-username]
export ORG_GRADLE_PROJECT_ossrhToken=[token]
export ORG_GRADLE_PROJECT_signingKey=$(cat [private-key-file.asc])
export ORG_GRADLE_PROJECT_signingPassword=[redacted]
```

Once you load these environment variables&mdash;and you have updated the `version` to a non-snapshot version&mdash;you
can create the staging repository via `./gradlew publish`.

You will need to close and release the staging repository on OSSRH
([https://s01.oss.sonatype.org/](https://s01.oss.sonatype.org/)), though everything is fairly intuitive:

- To find your staging repository, click the "Staging Repositories" link on the left sidebar.
- The UI for staging repositories contains "Close" and "Release" buttons.
- If anything goes wrong, you can also drop a staging repository using a similar "Drop" button.

The release will be typically be available within a half-hour, though it may take up to four hours
before it shows up in a search on [https://central.sonatype.com](https://central.sonatype.com).

### Step 9: Stage Future Releases Using a GitHub Workflow

Instead of running `./gradlew publish` manually, you can use a GitHub workflow like this to automate this step:

<!-- {% raw %} -->
```yaml
name: Publish
on:
  release:
    types:
      - created

jobs:
  publish:
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout
        uses: actions/checkout@v4.1.1

      - name: Setup Java
        uses: actions/setup-java@v4.0.0
        with:
          java-version: 21
          distribution: temurin

      - name: Validate Gradle wrapper
        uses: gradle/wrapper-validation-action@v1.1.0

      - name: Publish with Gradle
        uses: gradle/gradle-build-action@v2.11.1
        with:
          arguments: publish
        env:
          ORG_GRADLE_PROJECT_ossrhUsername: ${{ secrets.OSSRH_USERNAME }}
          ORG_GRADLE_PROJECT_ossrhToken: ${{ secrets.OSSRH_TOKEN }}
          ORG_GRADLE_PROJECT_signingKey: ${{ secrets.SIGNING_KEY }}
          ORG_GRADLE_PROJECT_signingPassword: ${{ secrets.SIGNING_PASSWORD }}
```
<!-- {% endraw %} -->

Notes:

- To use this workflow, you will also need to create corresponding [GitHub secrets][github-secrets] for your repository.
- This workflow will be triggered when you create a release in GitHub;
  Github releases are much simpler than Maven Central releases.
- You will still have to manually close and release the staging repository in OSSRH, though that part is pretty simple.
    - If you really want to automate those last two steps, you can explore the [Gradle Nexus Publish Plugin][plugin].

## Complete Example

Source: [https://github.com/mikewacker/drift](https://github.com/mikewacker/drift)

Here is the full convention plugin (`io.github.mikewacker.drift.publish-conventions.gradle.kts`):

```kotlin
plugins {
    java
    `maven-publish`
    signing
}

java {
    withSourcesJar()
    withJavadocJar()
}

tasks.javadoc {
    (options as StandardJavadocDocletOptions).addBooleanOption("Werror", true)
}

group = "io.github.mikewacker.drift"
version = "0.2.0-SNAPSHOT"

publishing {
    publications {
        create<MavenPublication>("mavenJava") {
            from(components["java"])
            versionMapping {
                usage("java-api") {
                    fromResolutionOf("runtimeClasspath")
                }
                usage("java-runtime") {
                    fromResolutionResult()
                }
            }
            pom {
                url = "https://github.com/mikewacker/drift"
                licenses {
                    license {
                        name = "MIT License"
                        url = "https://opensource.org/license/mit/"
                    }
                }
                developers {
                    developer {
                        id = "mikewacker"
                        name = "Mike Wacker"
                        email = "11431865+mikewacker@users.noreply.github.com"
                        url = "https://github.com/mikewacker"
                    }
                }
                scm {
                    connection = "scm:git:git://github.com/mikewacker/drift.git"
                    developerConnection = "scm:git:ssh://github.com:mikewacker/drift.git"
                    url = "https://github.com/mikewacker/drift/tree/main"
                }
            }
        }
    }
    repositories {
        maven {
            name = "ossrh"
            url = uri("https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/")
            credentials {
                val ossrhUsername: String? by project
                val ossrhToken: String? by project
                username = ossrhUsername
                password = ossrhToken
            }
        }
    }
}

signing {
    val signingKey: String? by project
    val signingPassword: String? by project
    useInMemoryPgpKeys(signingKey, signingPassword)
    sign(publishing.publications["mavenJava"])
}
```

Here is an example of the snippet that you must add to `build.gradle.kts` for each published module:

```kotlin
publishing {
    publications.named<MavenPublication>("mavenJava") {
        pom {
            name = "Drift"
            description = "Lightweight, all-in-one library to rapidly prototype a JSON API."
        }
    }
}
```

Here is the GitHub workflow for publishing (`.github/workflows/publish.yml`):

<!-- {% raw %} -->
```yaml
name: Publish
on:
  release:
    types:
      - created

jobs:
  publish:
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout
        uses: actions/checkout@v4.1.1

      - name: Setup Java
        uses: actions/setup-java@v4.0.0
        with:
          java-version: 21
          distribution: temurin

      - name: Validate Gradle wrapper
        uses: gradle/wrapper-validation-action@v1.1.0

      - name: Publish with Gradle
        uses: gradle/gradle-build-action@v2.11.1
        with:
          arguments: publish
        env:
          ORG_GRADLE_PROJECT_ossrhUsername: ${{ secrets.OSSRH_USERNAME }}
          ORG_GRADLE_PROJECT_ossrhToken: ${{ secrets.OSSRH_TOKEN }}
          ORG_GRADLE_PROJECT_signingKey: ${{ secrets.SIGNING_KEY }}
          ORG_GRADLE_PROJECT_signingPassword: ${{ secrets.SIGNING_PASSWORD }}
```
<!-- {% endraw %} -->

## References

Here are various sources of documentation that I used to both publish my own project and write this guide:

- Gradle: [Maven Publish Plugin][gradle-publish]
- Gradle: [The Signing Plugin][gradle-signing]
- GitHub: [Publishing Java packages with Gradle][github-publish]
- Central Repository: [Getting started][central-getting-started]
    - ...but, their [Gradle-specific documentation][central-gradle] appears to be very out-of-date.
- Central Repository: [GPG][central-gpg]

[central-coordinates]: https://central.sonatype.org/publish/requirements/coordinates/
[central-early-access]: https://central.sonatype.org/publish-ea/publish-ea-guide/
[central-getting-started]: https://central.sonatype.org/publish/publish-guide/
[central-gpg]: https://central.sonatype.org/publish/requirements/gpg/#create-a-ticket-with-sonatype
[central-gradle]: https://central.sonatype.org/publish/publish-gradle/
[central-token]: https://s01.oss.sonatype.org/#profile;User%20Token
[github-publish]: https://docs.github.com/en/actions/publishing-packages/publishing-java-packages-with-gradle
[github-secrets]: https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions
[gradle-publish]: https://docs.gradle.org/current/userguide/publishing_maven.html
[gradle-signing]: https://docs.gradle.org/current/userguide/signing_plugin.html
[joda-javadoc]: https://blog.joda.org/2012/11/javadoc-coding-standards.html
[oracle-javadoc]: https://www.oracle.com/technical-resources/articles/java/javadoc-tool.html
[plugin]: https://github.com/gradle-nexus/publish-plugin
