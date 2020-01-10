# GitHub Actions for Maven projects

List of GitHub Actions workflows that could be reused in various Maven (Java) projects that helps write well tested, high quality code. It was tested on Spring Boot applications, but it should be also working with other frameworks, like JavaEE.

## Usage

This folder consists two workflows:

* **Feature Branch Workflow** (*maven-feature.yml*)
* **Master Branch Workflow** (*maven-master.yml*)

If you add both of these files to your Java Maven project it will trigger GitHub Actions workflows, depending on which branch you commit and push new lines of code.

If you commit and push code on *master* branch in your project it will trigger the **Master Branch Workflow**. If you push the code to any other branch other then *master*, e.g. *feature/new-page*, and which's name doesn't start with *release* in it's name (this release workflow is not ready yet)it'll trigger the **Feature Branch Workflow**.

### Feature Branch Workflow

![feature-workflow.png](https://github.com/wkrzywiec/MyGitHubActions/blob/master/maven/pics/feature-workflow.png)

The main goal of this workflow is to check if the code is compiling and then run all unit tests.

```yaml

name: Feature Branch

on:
  push:
    branches:
      - '*'
      - '!master'
      - '!release'

jobs:

  unit-test:
    name: Unit Test
    runs-on: ubuntu-18.04

    steps:
      - uses: actions/checkout@v1
      - name: Set up JDK 11
        uses: actions/setup-java@v1
        with:
          java-version: 1.11
      - name: Maven Package
        run: mvn -B clean package -DskipTests
      - name: Maven Verify
        run: mvn -B clean verify

```

### Master Branch Workflow

![master-workflow.png](https://github.com/wkrzywiec/MyGitHubActions/blob/master/maven/pics/master-workflow.png)

This one is more complex. It has two stages - **Test** and **Publish**. Until first one is not finished second will not be triggered.

In **Test** stage there are two parallel jobs running:

* Running unit and integration tests
* Running unit tests and sending results to [sonarcloud](https://sonarcloud.io/) (static analysis tool)

In **Publish** stage there are also two parallel jobs running:

* Creating and publishing software artifact into [GitHub Packages](https://github.com/features/packages) so it can be than used as a dependency in other projects.
* Creating and publishing Docker image into Docker Hub so it can be than deployed on any machine or Cloud provider.

```yaml
name: Master Branch

on:
  push:
    branches:
      - 'master'

jobs:

  test:
    name: Test - Units & Integrations
    runs-on: ubuntu-18.04

    steps:
      - uses: actions/checkout@v1
      - name: Set up JDK 11
        uses: actions/setup-java@v1
        with:
          java-version: 11.0.4
      - name: Maven Package
        run: mvn -B clean package -DskipTests
      - name: Maven Verify
        run: mvn -B clean verify -Pintegration-test

  sonar:
    name: Test - SonarCloud Scan
    runs-on: ubuntu-18.04

    steps:
      - uses: actions/checkout@v1
      - name: Set up JDK 11
        uses: actions/setup-java@v1
        with:
          java-version: 11.0.4
      - name: SonarCloud Scan
        run: mvn -B clean verify -Psonar -Dsonar.login=${{ secrets.SONAR_TOKEN }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          
  artifact:
    name: Publish - GitHub Packages
    runs-on: ubuntu-18.04
    needs: [test, sonar]

    steps:
      - uses: actions/checkout@v1
      - name: Set up JDK 11
        uses: actions/setup-java@v1
        with:
          java-version: 11.0.4
      - name: Publish artifact on GitHub Packages
        run: mvn -B clean deploy -DskipTests
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}


  docker:
    name: Publish - Docker Hub
    runs-on: ubuntu-18.04
    needs: [test, sonar]
    env:
      REPO: ${{ secrets.DOCKER_REPO }}

    steps:
      - uses: actions/checkout@v1
      - name: Set up JDK 11
        uses: actions/setup-java@v1
        with:
          java-version: 11.0.4
      - name: Login to Docker Hub
        run: docker login -u ${{ secrets.DOCKER_USER }} -p ${{ secrets.DOCKER_PASS }}
      - name: Build Docker image
        run: docker build -t $REPO:latest -t $REPO:${GITHUB_SHA::8} .
      - name: Publish Docker image
        run: docker push $REPO
```

### Example projects

This workflow is used in my projects, e.g.:

* [NoticeBoard](https://github.com/wkrzywiec/NoticeBoard)

### More to read

If you want more details about how this worklfows works go to my blog post:

* *TO BE ADDED SOON*

## Installation

In order to make it work in your project you need simply to add both files into the *.github/workflows* folder located in the root of your project. So it'll looks like follows:

```
project-name/
├── .github/
│   └── workflows/
│       ├── maven-feature.yml
│       └── maven-master.yml
│
├── pom.xml
│
```

## Configuration

In order to make it work you need to do a little bit more than just copy-pasting the worflows.

First of all, you need to add secrets to your project (more about GitHub secrets could be found in the [official documentation](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/creating-and-using-encrypted-secrets)):

* **SONAR_TOKEN** - your Sonar User Token (more information could be found in the [official documentation](https://sonarcloud.io/documentation/user-guide/user-token/))
* **DOCKER_USER** - your Docker Hub username
* **DOCKER_PASS** - your Docker Hub password
* **DOCKER_REPO** - full name of the Docker Hub repository where Docker image will be pushed - *username/project_name* (e.g. *wkrzywiec/noticeboard*)

Also you need to have an account on *sonarcloud* (for open-sourced project it's free) and created a project that is linked to your GitHub project.

Finally in the *pom.xml* file you need to add following plugin, profiles and distribution managment repository:

```xml
<build>
		<plugins>
			<plugin>
				<groupId>org.jacoco</groupId>
				<artifactId>jacoco-maven-plugin</artifactId>
				<version>0.8.5</version>
				<executions>
					<execution>
						<id>default-prepare-agent</id>
						<goals>
							<goal>prepare-agent</goal>
						</goals>
					</execution>
					<execution>
						<id>default-report</id>
						<phase>prepare-package</phase>
						<goals>
							<goal>report</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>

	<profiles>
		<profile>
			<id>integration-test</id>
			<build>
				<plugins>
					<plugin>
						<groupId>org.codehaus.mojo</groupId>
						<artifactId>build-helper-maven-plugin</artifactId>
						<version>3.0.0</version>
						<executions>
							<execution>
								<id>add-integration-test-sources</id>
								<phase>generate-test-sources</phase>
								<goals>
									<goal>add-test-source</goal>
								</goals>
								<configuration>
									<sources>
										<source>src/integration-test/java</source>
									</sources>
								</configuration>
							</execution>
						</executions>
					</plugin>
					<plugin>
						<groupId>org.apache.maven.plugins</groupId>
						<artifactId>maven-failsafe-plugin</artifactId>
						<version>3.0.0-M3</version>
						<executions>
							<execution>
								<id>failsafe-integration-tests</id>
								<phase>integration-test</phase>
								<goals>
									<goal>integration-test</goal>
									<goal>verify</goal>
								</goals>
								<configuration>
									<skipTests>false</skipTests>
								</configuration>
							</execution>
						</executions>
					</plugin>
				</plugins>
			</build>
		</profile>
		<profile>
			<id>sonar</id>
			<properties>
				<sonar.sources>.</sonar.sources>
				<sonar.inclusions>src/main/java/**,src/main/resources/**</sonar.inclusions>
				<sonar.projectKey>NoticeBoard</sonar.projectKey>
				<sonar.organization>wkrzywiec</sonar.organization>
				<sonar.host.url>https://sonarcloud.io</sonar.host.url>
			</properties>
			<activation>
				<activeByDefault>false</activeByDefault>
			</activation>
			<build>
				<plugins>
					<plugin>
						<groupId>org.sonarsource.scanner.maven</groupId>
						<artifactId>sonar-maven-plugin</artifactId>
						<version>3.7.0.1746</version>
						<executions>
							<execution>
								<phase>verify</phase>
								<goals>
									<goal>sonar</goal>
								</goals>
							</execution>
						</executions>
					</plugin>
				</plugins>
			</build>
		</profile>
	</profiles>

	<distributionManagement>
		<repository>
			<id>github</id>
			<name>NoticeBoard Simple CRUD application</name>
			<url>https://maven.pkg.github.com/wkrzywiec/NoticeBoard</url>
		</repository>
	</distributionManagement>

```

## License

The MIT License - 2020 - Wojciech Krzywiec
