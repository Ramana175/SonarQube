# SonarQube – Complete DevOps Practical Guide

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

Developer → Git Repository → CI/CD Pipeline → Build and Tests → SonarQube Scanner → SonarQube Server → Quality Gate → Deployment Decision

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

Save the file and exit. Keep the configuration file private because it contains database credentials.

### Step 9: Configure Linux requirements

SonarQube uses Elasticsearch internally, so the operating system must meet the requirements of the installed release.

Review the server's required kernel parameters, file limits, memory, CPU, and disk requirements before starting the service.

Check available memory and disk space:

```bash
free -h
df -h
```

Check the file descriptor limit:

```bash
ulimit -n
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

**Important:** This is an example service definition. Verify the startup method and JAR filename for your exact SonarQube release before using it. Do not blindly replace a working service configuration.

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

### Step 12: Access the web interface

Open the following address in your browser:

```text
http://<EC2-PUBLIC-IP>:9000
```

Replace the placeholder with your instance's public IP address.

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

Use a plugin version compatible with the installed SonarQube release. For production, use HTTPS and store credentials in your CI/CD platform's secret store.

### Step 4: Verify results

1. Wait for Maven to finish.
2. Open `devops-demo` in SonarQube.
3. Confirm that a new analysis has appeared.
4. Review bugs, vulnerabilities, code smells, and security hotspots.
5. Review the Quality Gate result.
6. Investigate any failed conditions.

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

## 14. Troubleshooting

### SonarQube page does not open

```bash
sudo systemctl status sonarqube
sudo ss -lntp | grep ':9000'
sudo journalctl -u sonarqube -n 100 --no-pager
```

Also check the EC2 security group and network configuration.

### SonarQube fails to start

- Check service logs.
- Check database connectivity.
- Check Java compatibility.
- Check file permissions.
- Check memory, disk space, and operating-system requirements.

### Maven analysis fails

- Verify the SonarQube URL.
- Verify the token.
- Verify Execute Analysis permission.
- Check the project key.
- Confirm that the server is running and reachable.

### Quality Gate fails

- Open the latest analysis.
- Identify the failed condition.
- Inspect the corresponding metric.
- Check whether coverage data was imported.
- Fix the underlying issue or review the threshold according to the project's policy.

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
