# SonarQube: Complete DevOps Practical Guide

A hands-on guide to installing SonarQube on AWS EC2, managing access and quality rules, and connecting it to a Maven project and a CI/CD pipeline.

## At a glance

**What this lab covers:** a SonarQube server on Ubuntu EC2 (with PostgreSQL) that analyzes a Maven project and reports bugs, vulnerabilities, code smells, security hotspots, and a Quality Gate result.

**Tech:** SonarQube Community Build `26.10.0.133078` · Java 21 · PostgreSQL · Apache Maven · Ubuntu on AWS EC2

**Skills practiced:** Linux service setup (systemd), database setup, users/groups/permissions, Quality Profiles and Quality Gates, token handling, Maven integration, CI/CD integration, troubleshooting.

**Related:** the full pipeline that uses this SonarQube server (GitHub → Jenkins → Maven → SonarQube → Nexus → Tomcat) is at <https://github.com/Ramana175/Jenkins->.

## Table of contents

1. [What Is SonarQube?](#1-what-is-sonarqube)
2. [Why Is SonarQube Important for DevOps Engineers?](#2-why-is-sonarqube-important-for-devops-engineers)
3. [SonarQube Architecture](#3-sonarqube-architecture)
4. [Environment Used for This Practical](#4-environment-used-for-this-practical)
5. [Install SonarQube on Ubuntu EC2](#5-install-sonarqube-on-ubuntu-ec2)
6. [SonarQube Users and Groups](#6-sonarqube-users-and-groups)
7. [Global Permissions and Project Permissions](#7-global-permissions-and-project-permissions)
8. [Project Visibility](#8-project-visibility)
9. [Quality Profiles](#9-quality-profiles)
10. [Quality Gates](#10-quality-gates)
11. [Permission Templates](#11-permission-templates)
12. [Run SonarQube Analysis on a Maven Project](#12-run-sonarqube-analysis-on-a-maven-project)
    - [12.1 Maven configuration options](#121-maven-configuration-options)
    - [12.2 A small story: how Maven and SonarQube are connected](#122-a-small-story-how-maven-and-sonarqube-are-connected)
13. [SonarQube in a CI/CD Pipeline](#13-sonarqube-in-a-cicd-pipeline)
14. [Troubleshooting](#14-troubleshooting)
15. [SonarQube Administration Checklist](#15-sonarqube-administration-checklist)
16. [Final Outcome](#16-final-outcome)

## 1. What Is SonarQube?

SonarQube is a code quality and code security analysis tool used to identify problems in source code before the application is deployed to production.

It helps developers and DevOps engineers identify:

- **Bugs:** Problems that can cause an application to behave incorrectly.
- **Vulnerabilities:** Security weaknesses in source code.
- **Code Smells:** Code that may be difficult to maintain.
- **Security Hotspots:** Code that requires a security review.
- **Code Duplication:** Repeated code that may be improved.
- **Code Coverage:** The amount of code exercised by automated tests, when coverage reports are available.

SonarQube supports continuous inspection of code and can be integrated with CI/CD pipelines.

## 2. Why Is SonarQube Important for DevOps Engineers?

A DevOps engineer helps ensure that application code is built, tested, analyzed, and delivered reliably.

SonarQube helps DevOps engineers:

1. Integrate code analysis into CI/CD pipelines.
2. Identify code quality and security problems early.
3. Configure Quality Gates to evaluate analysis results.
4. Manage users, groups, and permissions.
5. Configure Quality Profiles and coding rules.
6. Monitor analysis results and troubleshoot failures.
7. Manage tokens, access control, backups, and server health.
8. Prevent code from progressing through a pipeline when configured quality conditions are not met.

## 3. SonarQube Architecture

SonarQube consists of several important components.

| Component        | Purpose                                                                             |
| ---------------- | ----------------------------------------------------------------------------------- |
| SonarQube Server | Receives and processes code analysis results and displays them in the web interface |
| Scanner          | Analyzes source code and submits analysis results                                   |
| Database         | Stores SonarQube application data                                                   |
| Quality Profile  | Defines the active analysis rules for a programming language                        |
| Quality Gate     | Evaluates whether analysis results meet configured conditions                       |
| CI/CD Pipeline   | Automates build, test, analysis, and delivery steps                                 |

### Analysis workflow

```text
Developer → Git Repository → CI/CD Pipeline → Build and Tests
         → SonarQube Scanner → SonarQube Server → Quality Gate → Deployment Decision
```

## 4. Environment Used for This Practical

The learning environment uses:

- Cloud: Amazon EC2
- Operating System: Ubuntu
- Java: Java 21
- SonarQube: Community Build
- Database: PostgreSQL
- Build Tool: Maven
- Project: `devops-demo`

The SonarQube version used during the lab was `26.10.0.133078`. Check the requirements for the version you install before selecting Java, database, and operating-system configurations.

**Instance size:** SonarQube runs a web server, a compute engine, and a search engine (Elasticsearch) together. A very small instance (for example 1 GB of RAM) often fails to start or is killed for lack of memory. Check the hardware requirements for your version and choose an instance with enough memory.

## 5. Install SonarQube on Ubuntu EC2

### Step 1: Launch an EC2 instance

1. Launch an Ubuntu EC2 instance.
2. Select an instance size suitable for SonarQube and its database.
3. Configure SSH access.
4. Attach a security group.
5. Connect to the instance using SSH.

For a practical lab, avoid exposing the SonarQube interface to the entire internet. Restrict access to your own IP or a trusted network.

### Step 2: Update Ubuntu

```bash
sudo apt update
sudo apt upgrade -y
```

### Step 3: Install Java

Use the Java version supported by the SonarQube release you selected.

```bash
sudo apt install openjdk-21-jdk -y
```

Verify Java:

```bash
java -version
javac -version
```

### Step 4: Install PostgreSQL

Install a PostgreSQL version supported by your SonarQube release.

```bash
sudo apt install postgresql -y
```

Check the service:

```bash
sudo systemctl status postgresql
```

Enable PostgreSQL at startup:

```bash
sudo systemctl enable postgresql
```

### Step 5: Create the SonarQube database and user

Open the PostgreSQL shell:

```bash
sudo -u postgres psql
```

Create a database user and database. Replace the example password with a strong, unique password.

```sql
CREATE USER sonaruser WITH ENCRYPTED PASSWORD 'REPLACE_WITH_STRONG_PASSWORD';
CREATE DATABASE sonarqube OWNER sonaruser;
\q
```

Keep the database password private. Do not commit it to GitHub.

### Step 6: Download and extract SonarQube

Download the appropriate SonarQube Community Build archive using the version you selected. Transfer it to the EC2 instance if necessary.

Create the installation directory:

```bash
sudo mkdir -p /opt/sonarqube
```

Extract the downloaded archive into `/opt/sonarqube`.

After extraction, identify the actual versioned directory:

```bash
ls -l /opt/sonarqube
```

The directory structure may look like this:

```text
/opt/sonarqube/
└── sonarqube-<version>/
    ├── bin/
    ├── conf/
    ├── data/
    ├── logs/
    └── temp/
```

Use the actual extracted directory name in all subsequent commands.

### Step 7: Create a dedicated Linux user

SonarQube should not run as root.

```bash
sudo useradd --system --create-home --shell /bin/bash sonarqube
```

Set the installation directory ownership. Replace `<version>` with the actual directory name.

```bash
sudo chown -R sonarqube:sonarqube /opt/sonarqube/sonarqube-<version>
```

### Step 8: Configure database connectivity

Open the SonarQube configuration file:

```bash
sudo nano /opt/sonarqube/sonarqube-<version>/conf/sonar.properties
```

Configure the database connection using the database and username created earlier.

```properties
sonar.jdbc.username=sonaruser
sonar.jdbc.password=REPLACE_WITH_STRONG_PASSWORD
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
```

Save the file and exit. Keep the configuration file private because it contains database credentials. Restrict who can read it:

```bash
sudo chown sonarqube:sonarqube /opt/sonarqube/sonarqube-<version>/conf/sonar.properties
sudo chmod 640 /opt/sonarqube/sonarqube-<version>/conf/sonar.properties
```

### Step 9: Configure Linux requirements

SonarQube uses Elasticsearch internally, so the operating system must meet the requirements of the installed release.

Review the server's required kernel parameters, file limits, memory, CPU, and disk requirements before starting the service.

Check available memory and disk space:

```bash
free -h
df -h
```

Check the current limits:

```bash
ulimit -n
sysctl vm.max_map_count
sysctl fs.file-max
```

Sonatype's documentation has listed minimums for these values, such as `vm.max_map_count` of at least `524288` and `fs.file-max` of at least `131072`. **Confirm the exact numbers for your release** before applying them. If the values are too low, SonarQube often fails to start with an Elasticsearch error about `vm.max_map_count`.

To apply a value permanently (example, using the number above):

```bash
echo "vm.max_map_count=524288" | sudo tee /etc/sysctl.d/99-sonarqube.conf
sudo sysctl --system
```

Do not apply arbitrary kernel settings from an unrelated SonarQube version. Configure the values required by your installed release.

### Step 10: Configure systemd

Create a service file:

```bash
sudo nano /etc/systemd/system/sonarqube.service
```

Example configuration:

```ini
[Unit]
Description=SonarQube Service
After=network.target postgresql.service

[Service]
Type=simple
User=sonarqube
Group=sonarqube
WorkingDirectory=/opt/sonarqube/sonarqube-<version>
ExecStart=/usr/bin/java -Xms32m -Xmx32m -Djava.net.preferIPv4Stack=true -jar /opt/sonarqube/sonarqube-<version>/lib/sonar-application-<version>.jar
Restart=always
LimitNOFILE=131072
LimitNPROC=8192
TimeoutStartSec=300
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```

**Important:** This is an example service definition. Verify the startup method and JAR filename for your exact SonarQube release before using it. Sonatype's documentation also shows a unit that starts SonarQube through the `sonar.sh` script in the `bin` directory; use whichever method your release's documentation recommends. Do not blindly replace a working service configuration.

### Step 11: Start SonarQube

Reload systemd and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable sonarqube
sudo systemctl start sonarqube
```

Check service status:

```bash
sudo systemctl status sonarqube
```

Inspect recent logs:

```bash
sudo journalctl -u sonarqube -n 100 --no-pager
```

Check whether port 9000 is listening:

```bash
sudo ss -lntp | grep ':9000'
```

SonarQube can take a minute or two to start. Wait before concluding that it failed.

### Step 12: Access the web interface

Open the following address in your browser:

```text
http://<EC2-PUBLIC-IP>:9000
```

Replace the placeholder with your instance's public IP address. The public IP can change after an EC2 stop/start unless you use an Elastic IP, so copy the current one from the EC2 console.

If the page does not open, check the service status, logs, network configuration, and EC2 security group.

For a temporary lab, restrict TCP port `9000` to your current public IP. For production, use HTTPS and suitable network access controls.

### Step 13: Secure the administrator account

1. Sign in with the initial administrator credentials.
2. Change the default password immediately.
3. Create named user accounts for normal users.
4. Avoid using the administrator account for routine code analysis.
5. Keep passwords and tokens out of source control.

## 6. SonarQube Users and Groups

Users are individual accounts. Groups allow administrators to manage permissions for multiple users.

Example groups:

- `sonar-administrators`: Trusted users who need administrative access.
- `sonar-users`: Users who need access to permitted projects.

### Practical tasks

1. Open Administration.
2. Navigate to user and group management.
3. Create or inspect the user `devuser1`.
4. Confirm that `devuser1` belongs to `sonar-users`.
5. Keep ordinary users out of the administrator group unless required.
6. Sign in as the normal user to verify the assigned access.

## 7. Global Permissions and Project Permissions

### Global permissions

Global permissions control actions across the SonarQube instance.

Examples include:

- Administer System
- Administer Quality Gates
- Administer Quality Profiles
- Create Projects
- Execute Analysis, where available

Only grant administrative permissions to users who need them.

### Project permissions

Project permissions control actions on a specific project.

| Permission                   | Purpose                                 |
| ---------------------------- | --------------------------------------- |
| Browse                       | View a project                          |
| See Source Code              | View source code in the interface       |
| Administer Issues            | Manage issues where permitted           |
| Administer Security Hotspots | Manage hotspot review status            |
| Administer                   | Manage project settings and permissions |
| Execute Analysis             | Submit analysis results                 |

Permission names and availability can vary by version and edition.

### Practical tasks

1. Open the `devops-demo` project.
2. Open Project Settings and the Permissions section.
3. Review access for `sonar-users` and `sonar-administrators`.
4. Remove unnecessary administrative access.
5. Test the result by signing in as `devuser1`.
6. Refresh and recheck permissions after any changes.

## 8. Project Visibility

A project may be public or private, depending on the available configuration.

For a private project:

1. Open `devops-demo`.
2. Open the project's settings.
3. Set visibility to Private.
4. Save the change if required.
5. Sign in as a normal user and test access.

Private visibility does not replace permission management. Verify which users and groups can access the project.

## 9. Quality Profiles

A Quality Profile defines which coding rules are active for a language.

For example, a Java profile determines which Java rules are used when the project is analyzed.

### Practical tasks

1. Open Quality Profiles.
2. Select Java.
3. Create a profile named `DevOps-Java-Profile`.
4. Set `Sonar way core` as its parent if inheritance is supported by the selected profile.
5. Review active rules.
6. Associate the profile with `devops-demo`.
7. Verify the association.

The profile defines analysis rules; it does not itself scan the project.

## 10. Quality Gates

A Quality Gate evaluates analysis results against configured conditions.

A Quality Gate may check metrics such as:

- Issues
- Security Hotspots Reviewed
- Coverage
- Duplicated Lines
- Condition Coverage

### Practical tasks

1. Open Quality Gates.
2. Create `DevOps-Practice-Gate`.
3. Add appropriate conditions.
4. Save the gate.
5. Open `devops-demo`.
6. Assign `DevOps-Practice-Gate` to the project.
7. Run an analysis and inspect the result.

The thresholds used in a practice lab are examples, not universal production standards. Coverage conditions require coverage data to be generated and imported.

## 11. Permission Templates

A permission template is a reusable set of project permissions. A default template can be used when projects are created.

### Practical tasks

1. Open Administration → Security → Permission Templates.
2. Inspect the default template.
3. Review its name and description.
4. Understand the project key pattern.
5. Review permissions assigned to each group.
6. Keep project administration restricted.
7. Decide which users or service accounts need Execute Analysis.
8. Test the template on a new practice project.

Do not assume that changing a template automatically updates every existing project's permissions. Verify the permissions on each relevant project.

## 12. Run SonarQube Analysis on a Maven Project

A DevOps engineer commonly integrates SonarQube analysis into a Maven build or CI/CD pipeline.

### Prerequisites

- A working SonarQube server
- A Java/Maven project with `pom.xml`
- A compatible JDK and Maven installation
- A valid analysis token
- Network access from the machine running Maven to the SonarQube server

### Step 1: Create an analysis token

1. Sign in using the account that will run the analysis.
2. Open the account's security settings.
3. Generate a token.
4. Store it securely.
5. Ensure that the account has permission to analyze the intended project.

Never commit the token to GitHub.

### Step 2: Set the token in the shell

```bash
read -s -p "SonarQube token: " SONAR_TOKEN
echo
export SONAR_TOKEN
```

### Step 3: Run the Maven analysis

From the directory containing `pom.xml`, run:

```bash
mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
  -Dsonar.host.url=http://<EC2-PUBLIC-IP>:9000 \
  -Dsonar.token="$SONAR_TOKEN"
```

Replace the placeholder with your SonarQube URL.

Use a plugin version compatible with the installed SonarQube release, and pin it for repeatable builds (for example `org.sonarsource.scanner.maven:sonar-maven-plugin:<version>:sonar`; the CI/CD project in this portfolio pins `5.8.0.7211`). For production, use HTTPS and store credentials in your CI/CD platform's secret store.

The scanner can also read the token from the `SONAR_TOKEN` environment variable. If you exported it in Step 2, you can leave out `-Dsonar.token`, which keeps the token out of the process list.

### Step 4: Verify results

1. Wait for Maven to finish.
2. Open `devops-demo` in SonarQube.
3. Confirm that a new analysis has appeared.
4. Review bugs, vulnerabilities, code smells, and security hotspots.
5. Review the Quality Gate result.
6. Investigate any failed conditions.

### 12.1 Maven configuration options

Maven needs to know **where SonarQube is** and **who is submitting**. There are three common places for the address, and you only need one.

**Option 1: on the command line** (shown in Step 3).

```bash
-Dsonar.host.url=http://<SONARQUBE_HOST>:9000
```

**Option 2: inside the existing `<properties>` block of `pom.xml`.** Do not create a second `<properties>` block.

```xml
<properties>
    <sonar.host.url>http://SONARQUBE_HOST:9000</sonar.host.url>
    <!-- Optional: by default the project key is groupId:artifactId -->
    <sonar.projectKey>com.example:devops-demo</sonar.projectKey>
</properties>
```

**Option 3: in `~/.m2/settings.xml`, so every Maven project on that machine uses it.**

```xml
<profiles>
  <profile>
    <id>sonar</id>
    <activation>
      <activeByDefault>true</activeByDefault>
    </activation>
    <properties>
      <sonar.host.url>http://SONARQUBE_HOST:9000</sonar.host.url>
    </properties>
  </profile>
</profiles>
```

Rules:

- The **address** can be in `pom.xml` or `settings.xml`. It is not a secret.
- The **token** is a secret. Keep it in an environment variable, Jenkins Credentials, or another secret store. Never put it in `pom.xml` or GitHub.
- The **project key** is how SonarQube names the project. Keep it the same between runs, or SonarQube will treat it as a new project.

### 12.2 A small story: how Maven and SonarQube are connected

#### The story: the Teacher and the Homework

Maven is a **student** who finishes homework (the code). SonarQube is the **teacher** who checks it and gives marks (bugs, vulnerabilities, code smells, and a pass or fail from the Quality Gate).

For the homework to reach the teacher and be marked, the student needs four things.

**1. The teacher's room address. That is `sonar.host.url`.**
Without it, the student walks to the nearest room (`localhost`), which is the wrong place. The homework never arrives, even though it was done well.

**2. A permission slip. That is the token (`SONAR_TOKEN`).**
The teacher only accepts homework from students who have a slip. The slip is a secret. If you post it in a public place (GitHub), a stranger can hand in homework with your name. So it stays in a locked box (an environment variable or Jenkins Credentials).

**3. A name on the folder. That is the project key.**
The teacher keeps one folder per project. By default the name is `groupId:artifactId`. If the name changes, the teacher starts a new folder and the old marks are not connected.

**4. A way to get the result. That is the Quality Gate.**
The teacher gives a **pass or fail** mark. With `-Dsonar.qualitygate.wait=true`, the student waits outside the room for the mark, and the build fails when the mark is a fail. Without it, the student hands in the homework and leaves, so the build can succeed even when the gate fails.

#### What happens when you run the Maven command

```text
1. Student finishes the work        -> mvn clean verify (compile and tests)
2. Student takes the work to the teacher's room
                                    -> reads sonar.host.url (the address)
3. Student shows the permission slip
                                    -> reads SONAR_TOKEN (the secret)
4. Teacher opens the project folder -> uses the project key
5. Teacher checks the work          -> Quality Profile rules
6. Teacher gives pass or fail       -> Quality Gate result
7. Student sees the mark            -> only if sonar.qualitygate.wait=true
```

#### Where each part is configured

| What the student needs | Where it is configured | Story name |
|---|---|---|
| Where SonarQube is | `-Dsonar.host.url`, `pom.xml` `<properties>`, or `settings.xml` profile | The teacher's room address |
| Who is allowed to submit | `SONAR_TOKEN` (shell, Jenkins Credentials) | The permission slip |
| Which project it is | `sonar.projectKey` (default `groupId:artifactId`) | The name on the folder |
| Which rules are used | Quality Profile in SonarQube | The teacher's checklist |
| Pass or fail | Quality Gate in SonarQube | The final mark |
| Fail the build on a fail mark | `-Dsonar.qualitygate.wait=true` | Waiting outside for the mark |

#### Why we update this configuration

Because in a cloud lab, things change.

- The SonarQube server gets a **new public IP** after an EC2 stop/start, so the address must be updated.
- A **token** is regenerated, expires, or is revoked, so the secret must be updated.
- The project **groupId or artifactId** changes, so the project key changes too.
- A **new Quality Gate** is assigned, so the pass or fail result can change.

#### What breaks if we do not update

| If this is wrong | What you see |
|---|---|
| No address, or an old IP | `Connection refused`, a timeout, or the scan goes to `localhost` |
| Missing or wrong token | `Not authorized` or a `401` error |
| Account lacks Execute Analysis permission | `Insufficient privileges` or a `403` error |
| Project key changed | A new, empty project appears in SonarQube |
| Quality Gate fails and `qualitygate.wait` is not set | Build shows `BUILD SUCCESS`, but the gate is red in SonarQube |

#### Remember

1. The **address** can be in `pom.xml`. The **token** never goes there.
2. Keep the token in an environment variable or Jenkins Credentials.
3. Keep the **project key** stable.
4. After an IP change, update the address everywhere it is set.
5. `BUILD SUCCESS` means the scan ran. It does not mean the Quality Gate passed, unless you use `sonar.qualitygate.wait`.
6. Change one thing at a time, then run the analysis again to test.

## 13. SonarQube in a CI/CD Pipeline

A typical pipeline contains these stages:

1. Checkout source code from Git.
2. Set up Java and Maven.
3. Compile the application.
4. Run automated tests.
5. Run SonarQube analysis.
6. Evaluate the Quality Gate.
7. Continue to packaging or deployment according to the pipeline policy.

The exact configuration depends on whether the team uses Jenkins, GitHub Actions, GitLab CI, or another CI/CD platform.

A DevOps engineer should ensure that the pipeline uses a valid token, the correct project key, a reachable SonarQube server, and appropriate Quality Gate handling.

### Example: Jenkins pipeline stage

This is an illustrative outline. Adapt the credential ID, server address, and Maven setup to your environment. The token is stored in Jenkins Credentials as a **Secret text** credential.

```groovy
pipeline {
    agent any

    environment {
        SONAR_HOST_URL = 'http://YOUR_SONARQUBE_HOST:9000'
    }

    stages {
        stage('SonarQube analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        mvn clean verify \
                          org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                          -Dsonar.host.url="$SONAR_HOST_URL" \
                          -Dsonar.qualitygate.wait=true
                    '''
                }
            }
        }
    }
}
```

Notes:

- The shell block uses single-style quotes (`'''`), so the secret is expanded by the shell and not copied into the Groovy script.
- `-Dsonar.qualitygate.wait=true` makes the build fail when the Quality Gate fails. Without it, a failed gate does not stop the pipeline.
- Never print the token in the build log.

## 14. Troubleshooting

### SonarQube page does not open

```bash
sudo systemctl status sonarqube
sudo ss -lntp | grep ':9000'
sudo journalctl -u sonarqube -n 100 --no-pager
```

Also check the EC2 security group and network configuration, and check that the public IP has not changed.

### SonarQube fails to start

- Check service logs.
- Check database connectivity.
- Check Java compatibility.
- Check file permissions.
- Check memory, disk space, and operating-system requirements.
- If the log mentions `vm.max_map_count` being too low, raise it as described in Step 9.
- If the process is killed with no clear error, the instance may be too small for SonarQube.

### Maven analysis fails

- Verify the SonarQube URL.
- Verify the token.
- Verify Execute Analysis permission.
- Check the project key.
- Confirm that the server is running and reachable.
- If you changed nothing but it stopped working after a stop/start, the SonarQube IP has probably changed. Update `sonar.host.url`.

### Quality Gate fails

- Open the latest analysis.
- Identify the failed condition.
- Inspect the corresponding metric.
- Check whether coverage data was imported.
- Fix the underlying issue or review the threshold according to the project's policy.

### Build succeeds but SonarQube shows a failed Quality Gate

- This is expected when the scanner is not told to wait for the result.
- Add `-Dsonar.qualitygate.wait=true` so the build fails with the gate.

### Quick reference

| Message or symptom | Likely cause | What to check |
|---|---|---|
| `Not authorized` or `401` | Wrong, expired, or missing token | Regenerate the token and update where it is stored |
| `403` or `Insufficient privileges` | The account cannot analyze this project | Execute Analysis permission on the project |
| `Connection refused` or timeout | Old IP, port closed, or service stopped | Current IP, security group on port `9000`, service status |
| A new empty project appears | Project key changed | `sonar.projectKey`, or `groupId` and `artifactId` |
| Coverage shows 0% | No coverage report imported | Coverage tool setup, such as a coverage plugin |

## 15. SonarQube Administration Checklist

- [ ] Install and start SonarQube.
- [ ] Verify Java and database compatibility.
- [ ] Secure the administrator account.
- [ ] Create and manage users and groups.
- [ ] Configure global permissions.
- [ ] Configure project permissions.
- [ ] Set project visibility.
- [ ] Create and assign a Quality Profile.
- [ ] Create and assign a Quality Gate.
- [ ] Review permission templates.
- [ ] Generate and secure an analysis token.
- [ ] Run a real Maven analysis.
- [ ] Verify the analysis and Quality Gate result.
- [ ] Review logs and troubleshoot a failure.
- [ ] Protect credentials and restrict network access.
- [ ] Plan backups, upgrades, and ongoing maintenance.

## 16. Final Outcome

After completing these practical exercises, a DevOps engineer should understand how to install and operate SonarQube, manage access, configure code-quality rules and Quality Gates, analyze a Maven project, troubleshoot common failures, and integrate code analysis into a CI/CD workflow.

**Next step:** use this SonarQube server from a Jenkins pipeline together with Nexus and Tomcat: <https://github.com/Ramana175/Jenkins->.
