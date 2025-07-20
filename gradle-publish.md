# Gradle: Publish a Java Project to Maven Central

You've consumed many third-party libraries from Maven Central. What if you want to publish a library to Maven Central?

- This guide provides step-by-step instructions, providing references for each step.
- Mistakes happen. This guide shows you how to verify each step&mdash;instead of hoping that everything "just works" at the end.
- This guide also covers automating releases with a GitHub workflow.

> NOTE:
> This guide uses Gradle 8 with the Kotlin DSL.

## 1. Fulfill the Requirements

To publish to Maven Central, your project must fulfill certain requirements.

- We will create (most of) a `buildSrc` plugin that fulfills these requirements. (e.g., `io.github.mikewacker.darc.java-publish.gradle.kts`)
    - The only part missing will be the `publications > repositories` block. This part will be added later.
- To publish a project, replace the `java-library` plugin with this plugin. (e.g., `id("io.github.mikewacker.darc.java-publish")`)

Let's start with the `plugins` block of our plugin:

```kotlin
plugins {
    `java-library`
    `maven-publish`
    signing
}
```

### Supply Javadoc and Sources

Add this code to your plugin:

```kotlin
java {
    withSourcesJar()
    withJavadocJar()
}

tasks.javadoc {
    (options as StandardJavadocDocletOptions).addBooleanOption("Werror", true)
    if (JavaVersion.current().isJava9Compatible) {
        (options as StandardJavadocDocletOptions).addBooleanOption("html5", true)
    }
}
```
- `Werror` is optional; this will enforce strict compliance with Javadoc standards.
    - On balance, these standards produce good documentation, even though they lead to some redundant documentation.
    - Since these standards only apply to the public API, they also encourage you to minimize your public API.
- You can safely remove the `if` statement if your project uses Java 9 or later.

**Verification**

1. `./gradlew build`
2. Verify that `build/libs` contains a sources JAR (`-sources.jar`) and a Javadoc JAR (`-javadoc.jar`)

**References**

- Maven Central: [Why do we have Requirements > Supply Javadoc and Sources](https://central.sonatype.org/publish/requirements/#supply-javadoc-and-sources)

### Required POM Metadata

**Required Metadata**

The POM must have the following metadata:

- correct coordinates (group, artifact, and version)
- project name, description, and URL
- license information
- developer information
- SCM information

**Create a Publication**

Add this code to your plugin, filling in the template:

> NOTE:
> This template is for a GitHub repository. The Maven Central documentation can help you adapt it to other options.

```kotlin
group = "[groupId]"
version = "[major].[minor].[patch]-SNAPSHOT"

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
                name = "[projectName]"
                description = "[projectDescription]"
                url = "https://github.com/[username]/[repository]"
                licenses {
                    license {
                        name = "[licenseName]"
                        url = "[licenseUrl]"
                    }
                }
                developers {
                    developer {
                        id = "[username]"
                        name = "[fullName]"
                        email = "[id]+[username]@users.noreply.github.com"
                        url = "https://github.com/[username]"
                    }
                }
                scm {
                    connection = "scm:git:git://github.com/[username]/[repository].git"
                    developerConnection = "scm:git:ssh://github.com:[username]/[repository].git"
                    url = "https://github.com/[username]/[repository]/tree/main"
                }
            }
        }
    }
}
```

- The group ID must match a registered namespace. (We will explain how to register namespaces later.)
    - E.g., if I register the `io.github.mikewacker` namespace, I can use `io.github.mikewacker.darc` as a group ID.
- The artifact ID is implicitly set to the project name.
    - For a single-project build, the project name is the value of `rootProject.name` in `settings.gradle.kts`.
    - For a multi-project build, the project name is the value in the `include()` call in `settings.gradle.kts`.
    - (In a multi-project build, the project path can be [modified](https://docs.gradle.org/current/userguide/multi_project_builds.html#modifying_a_subproject_path) if it doesn't match the project name.)
- For dependencies, the `versionMapping` block uses the versions that were resolved during the build.
    - (By default, the `maven-publish` plugin uses the versions that were declared in the `dependencies` block.)

**Verification**

1. `./gradlew clean generatePomFileForMavenJavaPublication`
2. Check the contents of the POM in `build/publications/mavenJava/pom-default.xml`.

When you build the code, the libraries will also have version numbers in their filenames (e.g., `artifact-1.0.0.jar`).

**References**

- Maven Central: [Why do we have Requirements > Required POM Metadata](https://central.sonatype.org/publish/requirements/#required-pom-metadata)
- Maven Central: [Coordinates](https://central.sonatype.org/publish/requirements/coordinates/)
- Gradle: [The Maven Publish Plugin > Publications > Customizing dependencies versions](https://docs.gradle.org/current/userguide/publishing_maven.html#publishing_maven:resolved_dependencies)

### Sign Files with GPG/PGP

**Generate and Distribute a GPG Key**

1. Run `gpg --gen-key`. You will be prompted for a real name, email address, and passphrase.
2. Run `gpg -k` to find the key ID of your key (e.g., `CA925CD6C9E8D064FF05B4728190C4130ABA0F98`).
3. Run `gpg --keyserver keyserver.ubuntu.com --send-keys [key-id]` to distribute your public key to a key server.

To verify that the key has been distributed, you can search for your key by its ID at <https://keyserver.ubuntu.com/>.

> NOTE:
> This key expires in 2 years, but you can extend the expiration and redistribute the updated key to the key server.

**Sign a Publication**

1. Add this code to your plugin:

```kotlin
signing {
    sign(publishing.publications["mavenJava"])
}
```

2. Export your key to a secret keyring file: `gpg --keyring secring.gpg --export-secret-keys > ~/.gnupg/secring.gpg`
3. Add these properties to `~/.gradle/gradle.properties`, filling in the template:
    - For `signing.keyId`, use the **last 8** symbols of the key ID.

```text
signing.keyId=[key-id-last-8]
signing.password=[passphrase]
signing.secretKeyRingFile=[home-dir]/.gnupg/secring.gpg
```

> WARNING:
> It may not be feasible to securely provide a keyring file to a CI server. We'll switch to an in-memory key later.

**Verification**

1. `./gradlew clean signMavenJavaPublication`
2. Verify that each file in `build/libs` has a corresponding signature file (`.asc`).

**References**

- Maven Central: [Why do we have Requirements > Sign Files with GPG/PGP](https://central.sonatype.org/publish/requirements/#sign-files-with-gpgpgp)
- Maven Central: [GPG](https://central.sonatype.org/publish/requirements/gpg/)
- Gradle: [The Signing Plugin > Signatory credentials](https://docs.gradle.org/current/userguide/signing_plugin.html#sec:signatory_credentials)

### Provide File Checksums

(The `maven-publish` plugin will automatically add checksum files when it publishes a project.)

**References**

- Maven Central: [Why do we have Requirements > Provide File Checksums](https://central.sonatype.org/publish/requirements/#provide-file-checksums)

## 2. Publish Manually

> IMPORTANT:
> OSSRH reached end of life on June 30, 2025. The `maven-publish` plugin does not yet support the new Portal API.

### Register a Namespace

1. [Create an account](https://central.sonatype.org/register/central-portal/) with the Central Portal. Use the option to log in with GitHub.
2. [Register a namespace](https://central.sonatype.org/register/namespace/) with the Central Portal.
    - If you log in via GitHub, your personal group ID (`io.github.[username]`) may already be registered.

**Verification**

1. In the Central Portal, check that your namespace is verified on the [namespaces](https://central.sonatype.com/publishing/namespaces) page.

### Publish to Maven Central

> NOTE:
> Don't worry about making mistakes. You can drop a deployment in the Central Portal if you make a mistake.

1. Add this code to the `publications` block of your plugin:

```kotlin
    repositories {
        maven {
            name = "ossrhStagingApi"
            url = uri("https://ossrh-staging-api.central.sonatype.com/service/local/staging/deploy/maven2/")
            credentials {
                val portalUsername: String? by project
                val portalPassword: String? by project
                username = portalUsername
                password = portalPassword
            }
        }
    }
```
2. Add this script (`publish-maven-central.sh`) to the root directory of your repository, filling in the template:

```shell
#!/usr/bin/env bash
# Publish to OSSRH Staging API.
./gradlew publish

# Transfer from OSSRH Staging API to Central Publisher Portal.
BEARER=$(printf "$ORG_GRADLE_PROJECT_portalUsername:$ORG_GRADLE_PROJECT_portalPassword" | base64)
curl --request POST \
    --include \
    --header "Authorization: Bearer $BEARER" \
    https://ossrh-staging-api.central.sonatype.com/manual/upload/defaultRepository/[namespace]
```

> IMPORTANT:
> For security reasons, `--include` is used instead of `--verbose`; the output will not show the `Authorization` header.

3. [Create a token](https://central.sonatype.org/publish/generate-portal-token/) in the Central Portal.
4. Add these environment variable to `~/.gradle/gradle.env`, filling in the template:

```shell
export ORG_GRADLE_PROJECT_portalUsername=[username]
export ORG_GRADLE_PROJECT_portalPassword=[password]
```

**Verification**

1. `source ~/.gradle/gradle.env`
2. `./publish-maven-central.sh`
3. Check the script output to ensure that `./gradlew publish` succeeded.
4. Check the script output to ensure that the `curl` request succeeded with a 200 response code.
5. In the Central Portal, check that a validated deployment is shown on the [deployments](https://central.sonatype.com/publishing/deployments) page.

From here, you can either drop or publish the deployment.

> NOTE:
> This workflow does not support `-SNAPSHOT` releases.

**References**

- Maven Central: [Publishing By Using the Portal OSSRH Staging API](https://central.sonatype.org/publish/publish-portal-ossrh-staging-api/)
- Maven Central: [OSSRH Sunset](https://central.sonatype.org/pages/ossrh-eol/)

## 3. Publish via a GitHub Workflow

### Switch to an In-Memory Signing Key

1. Replace the `signing` block of your plugin with this code:

```kotlin
signing {
    val signingKey: String? by project
    val signingPassword: String? by project
    useInMemoryPgpKeys(signingKey, signingPassword)
    sign(publishing.publications["mavenJava"])
}
```

2. Export an in-memory key: `gpg --armor --export-secret-keys [keyId] > ~/.gradle/seckey.asc`
3. Add these environment variables to `~/.gradle/gradle.env`, filling in the template:

```shell
export ORG_GRADLE_PROJECT_signingKey=$(cat ~/.gradle/seckey.asc)
export ORG_GRADLE_PROJECT_signingPassword=[passphrase]
```

**Verification**

1. `source ~/.gradle/gradle.env`
2. `./gradlew clean signMavenJavaPublication`
3. Verify that each file in `build/libs` has a corresponding signature file (`.asc`).

**References**

- Gradle: [The Signing Plugin > Signatory credentials > Using in-memory ascii-armored keys](https://docs.gradle.org/current/userguide/signing_plugin.html#sec:in-memory-keys)

### Create a GitHub Workflow

We will create a GitHub workflow that will be triggered when a GitHub release is created.

1. Add this workflow (`.github/workflows/publish.yml`) to your repository (updating action versions as you see fit):

```yaml
name: Publish
on:
  release:
    types: [created]
  workflow_dispatch:

jobs:
  publish:
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout
        uses: actions/checkout@v4.2.2

      - name: Setup Java
        uses: actions/setup-java@v4.7.1
        with:
          java-version: 21
          distribution: temurin

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4.4.1

      - name: Publish
        run: ./publish-maven-central.sh
        env:
          ORG_GRADLE_PROJECT_signingKey: ${{ secrets.SIGNING_KEY }}
          ORG_GRADLE_PROJECT_signingPassword: ${{ secrets.SIGNING_PASSWORD }}
          ORG_GRADLE_PROJECT_portalUsername: ${{ secrets.PORTAL_USERNAME }}
          ORG_GRADLE_PROJECT_portalPassword: ${{ secrets.PORTAL_PASSWORD }}
```

2. Navigate to the secrets page of your repository: Settings > Secrets and variables > Actions
3. Add the following repository secrets (not environment secrets): `SIGNING_KEY`, `SIGNING_PASSWORD`, `PORTAL_USERNAME`, and `PORTAL_PASSWORD`.
    - For `SIGNING_KEY`, copy and paste the contents of `~/.gradle/seckey.asc`.

**Verification**

1. Create a release in GitHub.
2. In the Actions tab of the repository, check the workflow result.
3. In the Central Portal, check that a validated deployment is shown on the [deployments](https://central.sonatype.com/publishing/deployments) page.

> IMPORTANT:
> The workflow may "succeed" if `./publish-maven-central.sh` fails; you will have to examine the output for this action.

> NOTE:
> Since the workflow includes `workflow_dispatch`, you can rerun it manually if something goes wrong.

**References**

- GitHub: [Publishing Java packages with Gradle](https://docs.github.com/en/actions/tutorials/publishing-packages/publishing-java-packages-with-gradle)
- GitHub: [Using secrets in GitHub Actions](https://docs.github.com/en/actions/how-tos/writing-workflows/choosing-what-your-workflow-does/using-secrets-in-github-actions)

## Appendix: Complete Example

Source: <https://github.com/mikewacker/darc>

`buildSrc/src/main/kotlin/io.github.mikewacker.darc.java-publish.gradle.kts`:

```kotlin
plugins {
    `java-library`
    `maven-publish`
    signing
}

tasks.compileJava {
    options.release = 11
}

java {
    withSourcesJar()
    withJavadocJar()
}

tasks.javadoc {
    (options as StandardJavadocDocletOptions).addBooleanOption("Werror", true)
    (options as StandardJavadocDocletOptions).addBooleanOption("html5", true)
}

group = "io.github.mikewacker.darc"
version = "0.1.1"

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
                name = "DARC"
                description = "Dagger And Request Context (for Dropwizard)"
                url = "https://github.com/mikewacker/darc"
                licenses {
                    license {
                        name = "MIT License"
                        url = "https://opensource.org/license/mit"
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
                    connection = "scm:git:git://github.com/mikewacker/darc.git"
                    developerConnection = "scm:git:ssh://github.com:mikewacker/darc.git"
                    url = "https://github.com/mikewacker/darc/tree/main"
                }
            }
        }
    }
    repositories {
        maven {
            name = "ossrhStagingApi"
            url = uri("https://ossrh-staging-api.central.sonatype.com/service/local/staging/deploy/maven2/")
            credentials {
                val portalUsername: String? by project
                val portalPassword: String? by project
                username = portalUsername
                password = portalPassword
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

`publish-maven-central.sh`:

```shell
#!/usr/bin/env bash
# Publish to OSSRH Staging API.
./gradlew publish

# Transfer from OSSRH Staging API to Central Publisher Portal.
BEARER=$(printf "$ORG_GRADLE_PROJECT_portalUsername:$ORG_GRADLE_PROJECT_portalPassword" | base64)
curl --request POST \
    --include \
    --header "Authorization: Bearer $BEARER" \
    https://ossrh-staging-api.central.sonatype.com/manual/upload/defaultRepository/io.github.mikewacker
```

`.github/workflows/publish.yml`:

```yaml
name: Publish
on:
  release:
    types: [created]
  workflow_dispatch:

jobs:
  publish:
    runs-on: ubuntu-24.04
    steps:
      - name: Checkout
        uses: actions/checkout@v4.2.2

      - name: Setup Java
        uses: actions/setup-java@v4.7.1
        with:
          java-version: 21
          distribution: temurin

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4.4.1

      - name: Publish
        run: ./publish-maven-central.sh
        env:
          ORG_GRADLE_PROJECT_signingKey: ${{ secrets.SIGNING_KEY }}
          ORG_GRADLE_PROJECT_signingPassword: ${{ secrets.SIGNING_PASSWORD }}
          ORG_GRADLE_PROJECT_portalUsername: ${{ secrets.PORTAL_USERNAME }}
          ORG_GRADLE_PROJECT_portalPassword: ${{ secrets.PORTAL_PASSWORD }}
```
