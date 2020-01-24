# Integrating Github Actions for Kotlin Multiplatform


With Kotlin Multiplatform (KMP) it's possible to build artifacts for multiple platforms using the same toolchain, 
but until portable artifacts for KMP are released you need to build platform artifacts on
their respective platforms.

With SqlDelight 1.2.2 we now also deploy Windows (mingW) artifacts, meaning it's impossible to publish
from a single OS (since we also support macOS targets). There was no simple setup with Travis and Github Actions
is the new hot stuff so we gave that a go and here's how it works:

### Continuous Integration

Github Actions is configured through yaml files, each one describing a specific workflow. For CI we have a few workflows but the main one
is [test](https://github.com/cashapp/sqldelight/blob/master/.github/workflows/PR.yml) which is a workflow configured to run on PRs.

```yaml
on:
  pull_request:
    paths-ignore:
      - 'docs/**'
      - '*.md'
```

It ignores changes to documentation meaning PRs that only touch the ignored files immediately go green. Dope!

In a workflow file you specify a series of jobs that happen when the workflow is triggered, and you can use
this to boot up all the different OS's you target for KMP. You can do it by specifying multiple jobs (one for each OS)
or using a build matrix which is what we do.

```yaml
jobs:
  build:
    strategy:
      matrix:
        os: [macOS-latest, windows-latest, ubuntu-latest]
        job: [instrumentation, test]
        exclude:
          - os: windows-latest
            job: instrumentation
          - os: ubuntu-latest
            job: instrumentation
```

This configures 6 jobs: `macOS-latest/test`, `macOS-latest/instrumentation`, `windows-latest/test`, `windows-latest/instrumentation`, 
`ubuntu-latest/test` and `ubuntu-latest/instrumentation` (its the cross product of matrix properties), 
but we only run instrumentation on macOS so we ignore the other 2
instrumentation jobs and we're down to 4. Since we want each machine to run a different gradle task we 
can switch on the matrix property for that specific job and skip the step if its not the right configuration:

```yaml
- name: Run ios tests
  if: matrix.os == 'macOS-latest' && matrix.job == 'test'
  run: ./gradlew iosTest
```

The final piece to the puzzle is the thing that makes github actions so special: actions! Each step in the workflow
can be a simple shell command or you can use an action - of which theres already tons of useful ones on the 
[marketplace](https://github.com/marketplace?type=actions). For example the `macOS/instrumentation` job uses
the [Android Emulator Runner](https://github.com/marketplace/actions/android-emulator-runner) action:

```yaml
- name: Run instrumentation tests
  if: matrix.job == 'instrumentation'
  uses: reactivecircus/android-emulator-runner@v2
  with:
    api-level: 29
    script: ./gradlew connectedCheck :sqldelight-gradle-plugin:build
```

This is the game changer - not having to worry about maintaining your own emulator script is huge. And the same goes for
all the other actions available. One of the other workflows we have set up to run on PRs verifies the gradle wrapper
is a trusted published version to protect your repo from an attack vector - again it takes no effort to apply it since all the
work is done for you!

### Continuous Deployment

The more challenging bit is configuring releases. To do this we have a single workflow [Release](https://github.com/cashapp/sqldelight/blob/master/.github/workflows/Release.yml)
which publishes the artifacts and IntelliJ plugin - both for official releases and snapshots.

To facilitate releasing you need to give Github some
[secrets](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/creating-and-using-encrypted-secrets) - your
Sonatype username/password and a GPG key, and then expose those to the workflow job step which runs the publish:

```yaml
- name: Publish the windows artifact
  env:
    ORG_GRADLE_PROJECT_SONATYPE_NEXUS_USERNAME: ${{ secrets.ORG_GRADLE_PROJECT_SONATYPE_NEXUS_USERNAME }}
    ORG_GRADLE_PROJECT_SONATYPE_NEXUS_PASSWORD: ${{ secrets.ORG_GRADLE_PROJECT_SONATYPE_NEXUS_PASSWORD }}
    ORG_GRADLE_PROJECT_signingKey: ${{ secrets.ORG_GRADLE_PROJECT_signingKey }}
  run: ./gradlew publishMingwPublicationToMavenRepository
```

The GPG key you add to Github settings should be an ascii-armored version of the key which you can get
by running `gpg --export --ascii <keyid> | pbcopy` in your console. If you're using gradle's signing plugin 
(if you're not sure, you probably are) it needs to be configured to use the ascii armored key.

```groovy
def isReleaseBuild() {
  return VERSION_NAME.contains("SNAPSHOT") == false
}

def getGpgKey() {
  return hasProperty('signingKey') ? signingKey : ""
}

signing {
  required { isReleaseBuild() }

  def signingKey = getGpgKey()
  if (!signingKey.isEmpty() && isReleaseBuild()) {
    useInMemoryPgpKeys(signingKey, "")
    sign publishing.publications
  }
}
```

The KMP Gradle plugin is smart enough to not try and build/publish for unsupported architectures, so we just let the macOS
job publish all the artifacts it can, and then Windows only tries to publish the mingW publications.

---

And thats it! We also have a workflow to [deploy gh-pages with mkdocs](https://github.com/cashapp/sqldelight/blob/master/.github/workflows/Publish-Website.yml)
and a job which [publishes the IntelliJ plugin](https://github.com/cashapp/sqldelight/blob/master/.github/workflows/Release.yml#L37-L47). With Github Actions
we've been able to shrink the actual releasing process [pretty dramatically](https://github.com/cashapp/sqldelight/commit/5acee5551bd1c6a19233ed4b32c0c7bb445faff2)
and I'm looking forward to using it more!
