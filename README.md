# JenkinsInterviewQ&A

This repository provides a concise Q&A guide for preparing for Jenkins-related interview questions, focusing on Jenkins shared libraries, Maven build targets, artifact repositories, Artifactory configuration, CI troubleshooting, and production debugging. It is designed to help candidates understand key concepts and articulate answers effectively in a DevOps or CI/CD context.

## Q&A for Jenkins Interview

### 1. What are Jenkins Shared Libraries, and how do they work?

**Question**: Can you explain Jenkins shared libraries and their role in CI/CD pipelines?

**Answer**:
Jenkins shared libraries are reusable pieces of Groovy code stored in a version-controlled repository (e.g., GitHub) that multiple Jenkins pipelines can use. They are particularly useful in enterprise environments where multiple development teams share similar CI/CD logic, such as building, testing, or deploying applications.

- **How They Work**:
  - Shared libraries are stored in a Git repository with a specific folder structure:
    ```
    (root)
    ├── src/                 # Groovy source files (for classes)
    ├── vars/                # Global variables and reusable functions
    │   └── buildApp.groovy  # Example: Custom function for building apps
    └── resources/           # Non-Groovy files (e.g., scripts, configs)
    ```
  - The library is referenced in a `Jenkinsfile` using the `@Library` annotation:
    ```groovy
    @Library('my-shared-library@master') _
    pipeline {
        agent any
        stages {
            stage('Build') {
                steps {
                    script {
                        buildApp() // Calls a function from vars/buildApp.groovy
                    }
                }
            }
        }
    }
    ```
  - Jenkins loads the library from the specified repository and branch during pipeline execution.
  - If existing library functions meet the pipeline's needs, they are reused. If new logic is required, it can be added to the `Jenkinsfile` or contributed to the shared library for reuse across teams.

- **Benefits**:
  - Promotes code reuse, reducing duplication in `Jenkinsfile` configurations.
  - Simplifies maintenance by centralizing common logic.
  - Enables collaboration across teams with standardized pipeline steps.

### 2. What are five Maven build targets you frequently use, and what do they do?

**Question**: Name five Maven build targets you use regularly and explain their purpose.

**Answer**:
Maven is a widely used build tool for Java projects, and the following five build targets are commonly used in day-to-day development:

| Target | Command | Purpose | Location |
|--------|---------|---------|----------|
| **Clean** | `mvn clean` | Removes `target/` directory with previous build artifacts | `target/` |
| **Compile** | `mvn compile` | Compiles `src/main/java` to bytecode | `target/classes/` |
| **Test** | `mvn test` | Runs unit tests from `src/test/java` | `target/surefire-reports/` |
| **Package** | `mvn package` | Creates JAR/WAR artifact after compile+test | `target/*.jar` |
| **Install** | `mvn install` | Installs artifact to local repo (`~/.m2`) or remote (JFrog/Nexus) | `~/.m2/repository/` |

**Example**: 
```bash
mvn clean compile test package install
```

### 3. Which artifact repository do you use for builds, and what is its purpose?

**Question**: What artifact repository do you use for managing build artifacts, and why is it important?

**Answer**:
Artifact repositories store and manage build artifacts like JAR, WAR, or EAR files, enabling versioned access for developers and CI/CD pipelines.

- **Repositories Used**: **Nexus** and **JFrog Artifactory**
- **Purpose**:
  - **Storage & Versioning**: Stores artifacts (e.g., `my-app-1.2.3.jar`) for download
  - **Dependency Management**: Hosts 3rd-party dependencies from `pom.xml`
  - **Security**: Scans external JARs before internal storage
  - **CI/CD Integration**: Uploads during builds, downloads during deployments

**Workflow**:
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>5.3.10</version>
</dependency>
```
Dependencies download from Nexus/Artifactory → Build → Upload artifact.

### 4. How do you configure Artifactory for your application in Maven?

**Question**: Walk me through configuring Artifactory in Maven for your builds.

**Answer**:
Configuring Artifactory in Maven requires **two simple steps**:

#### Step 1: Global Configuration (`~/.m2/settings.xml`)
Configure repository locations under `<profiles>` and `<pluginRepositories>`:

```xml
<settings>
  <profiles>
    <profile>
      <id>artifactory</id>
      <repositories>
        <repository>
          <id>central</id>
          <url>https://mycompany.jfrog.io/artifactory/libs-release</url>
        </repository>
      </repositories>
      <pluginRepositories>
        <pluginRepository>
          <id>central</id>
          <url>https://mycompany.jfrog.io/artifactory/plugins-release</url>
        </pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>
  <activeProfiles>
    <activeProfile>artifactory</activeProfile>
  </activeProfiles>
</settings>
```

#### Step 2: Application-Specific (`pom.xml`)
Configure deployment for **two Artifactory instances** (release + snapshot):

```xml
<distributionManagement>
  <repository>
    <id>artifactory-release</id>
    <url>https://mycompany.jfrog.io/artifactory/libs-release</url>
  </repository>
  <snapshotRepository>
    <id>artifactory-snapshot</id>
    <url>https://mycompany.jfrog.io/artifactory/libs-snapshot</url>
  </snapshotRepository>
</distributionManagement>
```

**Deploy Command**: `mvn clean deploy` uploads to configured Artifactory.

### 5. Build passes locally but fails in CI. How will you troubleshoot?

**Question**: A Maven build works on your local machine but fails in Jenkins CI. How do you debug?

**Answer**:
**Systematic Troubleshooting Approach** (for Maven):

| Step | Action | Command/Check | Common Issue |
|------|--------|---------------|--------------|
| 1 | **Check Logs** | `Jenkins Console Output` | Compilation/Test errors |
| 2 | **Clear Cache** | `mvn clean` | Stale `target/` artifacts |
| 3 | **Environment Vars** | `echo $JAVA_HOME` | Missing `JAVA_HOME`, `PATH` |
| 4 | **File Permissions** | `ls -la target/` | Jenkins agent lacks write access |
| 5 | **OS Differences** | `uname -a` | Windows vs Linux line endings |

**Example Commands in Jenkins**:
```groovy
stage('Build') {
    steps {
        sh 'mvn clean package'  // Always clean first
    }
}
```

**Quick Fix**: Add `mvn clean` before `mvn package` in Jenkins pipeline.

### 6. CI pipeline succeeds, but the app is broken in production. Why?

**Question**: Your Jenkins pipeline passes all tests, but the application fails in production. How do you debug?

**Answer**:
**Production Debugging Workflow** (Dev → Staging → Prod):

| Issue | Symptoms | Troubleshooting | Solution |
|-------|----------|-----------------|----------|
| **Version Mismatch** | Works in staging, fails in prod | `kubectl get deploy -o yaml` | Align cluster versions |
| **Traffic Scale** | 1K req OK, 100K fails | `kubectl logs`, `kubectl top pods` | Performance testing (100K req) |
| **OOM/CrashLoop** | Pods restarting | `kubectl describe pod` | Increase resource limits |
| **Deployment Strategy** | Full rollout fails | Canary deployment | 10% → 30% → 100% rollout |
| **Build Flags** | Different prod config | Compare Jenkins stages | Standardize build args |

**Example Canary Deployment**:
```yaml
# Rollout 10% first
kubectl set image deploy/webapp webapp=v2 --replicas=1
# Monitor 2 days, then scale
kubectl scale deploy/webapp --replicas=3
```

**Performance Testing**:
```bash
# Load test with 100K requests
hey -n 100000 -c 100 http://prod-app.com
```

**Prevention**: 
- Add load testing stage in Jenkins
- Use canary/blue-green deployments
- Compare staging vs prod build flags

## Conclusion
Understanding Jenkins shared libraries, Maven configurations, Artifactory setup, CI troubleshooting, and production debugging is crucial for enterprise DevOps roles. This Q&A covers practical scenarios from code reuse to production reliability, preparing candidates for efficient pipeline development and maintenance in production-grade systems.
