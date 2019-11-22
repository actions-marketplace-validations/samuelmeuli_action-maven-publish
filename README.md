# Maven Publish GitHub Action

**GitHub Action for automatically publishing a Maven package to the Central Repository**

It's recommended to publish using the [Nexus Staging Maven Plugin](https://github.com/sonatype/nexus-maven-plugins/tree/master/staging/maven-plugin), which greatly simplifies the process.

## Requirements

The following instructions assume that you've already configured your repository for deployment to the Central Repository using the Nexus Staging Maven Plugin. If you haven't, you can read about how to do this [here](https://central.sonatype.org/pages/apache-maven.html#performing-a-release-deployment).

In the plugin's configuration, you will probably want to set the following option to automatically deploy to the Central Repository after successfully staging on Nexus:

```xml
<plugin>
  <groupId>org.sonatype.plugins</groupId>
  <artifactId>nexus-staging-maven-plugin</artifactId>
  <!-- ... -->
  <configuration>
    <!-- ... -->
    <autoReleaseAfterClose>true</autoReleaseAfterClose>
  </configuration>
</plugin>
```

Furthermore, make sure your GPG plugin is configured like this:

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-gpg-plugin</artifactId>
  <version>1.6</version>
  <executions>
    <execution>
      <id>sign-artifacts</id>
      <phase>verify</phase>
      <goals>
        <goal>sign</goal>
      </goals>
    <configuration>
        <!-- Prevent `gpg` from using pinentry programs -->
        <gpgArguments>
          <arg>--pinentry-mode</arg>
          <arg>loopback</arg>
        </gpgArguments>
      </configuration>
    </execution>
  </executions>
</plugin>
```

## Usage

### Authentication

In your project's GitHub repository, go to Settings → Secrets. On this page, set the following variables:

- `gpg_private_key`: The GPG base64-encoded private key you use to sign the published artifacts:
  - Run `gpg --list-secret-keys` and locate the key you'd like to use. Copy its ID
  - Export and encode the key with `gpg -a --export-secret-keys KEY_ID | base64` (and replace `KEY_ID` with the ID you copied)
- `gpg_passphrase`: The passphrase for the exported GPG key
- `ossrh_username`: Your Sonatype OSSRH username (not the email address!)
- `ossrh_password`: Your Sonatype OSSRH password

These secrets will be passed as environment variables into the action, allowing it to do the deployment for you.

### Action

Create a GitHub workflow file (e.g. `.github/workflows/release.yml`) in your repository. Use the following configuration, which tells GitHub to use the Maven Publish Action when running your CI pipeline. The steps are self-explanatory:

```yml
name: Release

# Run workflow only on commits to `master`
on:
  push:
    branches:
      - master

jobs:
  release:
    runs-on: ubuntu-18.04
    steps:
      - name: Check out Git repository
        uses: actions/checkout@v1

      - name: Release to Central Repository
        uses: samuelmeuli/action-maven-publish@master
        with:
          gpg_private_key: ${{ secrets.gpg_private_key }}
          gpg_passphrase: ${{ secrets.gpg_passphrase }}
          ossrh_username: ${{ secrets.ossrh_username }}
          ossrh_password: ${{ secrets.ossrh_password }}
```

This should be all the configuration you need. Every time you push to `master`, the action will be run. If your `pom.xml` file contains a non-snapshot version tag and all tests pass, your package will automatically be deployed to the Central Repository.

## Development

### Implementation

The Maven Publish GitHub Action works the following way:

- When imported from a CI workflow in your project, GitHub will look for this repository's [`action.yml`](./action.yml) file. This file tells GitHub to run a new Docker container for the action and to pass in the action's input variables (GPG key and OSSRH login credentials).
- Docker will spin up a new container with Java and Maven installed (see [`Dockerfile`](./Dockerfile)).
- In the container, the [`entrypoint.sh`](./entrypoint.sh) script will be executed. It checks whether all required variables are defined, decodes the GPG private key and runs the Maven deploy command. Maven will use this repository's [`settings.xml`](./settings.xml) file, which instructs it to use the GPG passphrase and OSSRH credentials from the provided environment variables.

### Contributing

Suggestions and contributions are always welcome! Please discuss larger changes via issue before submitting a pull request.