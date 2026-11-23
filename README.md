# 34_VIBE-

VIDE creation Projects:
    -Restassured
    -yaml_HOSTmachine(no POM for dependency creation)
        


--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
How does a company with no access to the internet automate the Java build process without maven

Automating Java builds in an air-gapped (offline) environment without Maven comes down to solving three problems: 
picking an alternative build tool, 
managing dependencies locally, 
running a self-hosted CI/CD server.

1. Choose an Alternative Build Tool
Without Maven, companies typically rely on one of four approaches depending on project size and complexity:

Build Tool	Best For	How It Works Offline
Gradle	Modern enterprise Java apps	Uses a local Gradle distribution (via the Gradle Wrapper pointing to an internal file server) and runs with the --offline flag against a local cache or internal repository.
Apache Ant	Legacy systems & strict air-gaps	Pure XML-based scripting (build.xml). Compiles directly against JARs stored in a local lib/ folder—zero network calls required.
Bazel	Large monorepos & high security	Designed for hermetic, reproducible builds. Dependencies are explicitly vendored or fetched from an internal HTTP mirror.
Make / Shell Scripts	Small utilities or minimal setups	Directly invokes javac, jar, and java commands in sequence. No extra build tool installation required.
2. Manage Dependencies Offline
Because the build server cannot reach Maven Central, you must bring third-party JARs inside the network through a controlled process:

Internal Artifact Repository (Recommended for scale): Host JFrog Artifactory or Sonatype Nexus on an internal server. A security or platform team downloads approved JARs on an internet-connected machine, scans them for vulnerabilities, transfers them across the air-gap (via approved media or a unidirectional security gateway), and uploads them to the internal repo. Gradle or Apache Ivy then pulls exclusively from this internal URL.

Vendoring (Checking JARs into Version Control): For simpler setups (especially with Ant or Make), commit all required .jar files directly into a lib/ directory inside your Git or SVN repository. Every developer and build agent gets the exact same dependencies simply by pulling the code.

3. Automate with Self-Hosted CI/CD
Run an on-premise automation server such as Jenkins, GitLab CE/EE, or TeamCity on your internal network:

Pre-bake your build agents: Create standardized VM templates or internal Docker images that already have the JDK, your chosen build tool (Ant/Gradle/Bazel), and any required plugins pre-installed.

Trigger builds from internal VCS: Point your CI server to your self-hosted Git/SVN repository so every commit triggers a build automatically.

Archive artifacts internally: Configure the CI pipeline to push the compiled .jar or .war files to an internal file share or your internal Nexus/Artifactory server for deployment.

Example: Completely Offline Ant Build
If you want the simplest possible setup with zero external tooling beyond the JDK and Ant, a build.xml file automates compilation, testing, and packaging using only local files:

XML
<project name="OfflineApp" default="package" basedir=".">
    <property name="src.dir" value="src"/>
    <property name="lib.dir" value="lib"/>
    <property name="build.dir" value="build"/>

    <path id="classpath">
        <fileset dir="${lib.dir}" includes="**/*.jar"/>
    </path>

    <target name="clean">
        <delete dir="${build.dir}"/>
    </target>

    <target name="compile" depends="clean">
        <mkdir dir="${build.dir}/classes"/>
        <javac srcdir="${src.dir}" destdir="${build.dir}/classes" classpathref="classpath" includeantruntime="false"/>
    </target>

    <target name="package" depends="compile">
        <jar destfile="${build.dir}/app.jar" basedir="${build.dir}/classes"/>
    </target>
</project>
Your internal Jenkins or GitLab Runner simply executes ant package on every commit—no internet connection or Maven installation needed.

To dive deeper into your specific setup:

Configure Gradle for fully offline air-gapped builds

Design a secure sneakernet workflow for importing JARs
What about when using eclipse as the IDE with jar files manually obtained from an artefactory

When developers use Eclipse with manually downloaded JARs, the biggest hurdle to automation is classpath drift—ensuring the CI server compiles with the exact same JARs that Eclipse uses, without relying on paths specific to a developer's machine (like C:\Users\jdoe\Downloads\).

Here is how to bridge a manual Eclipse setup into a reliable automated build.

1. Standardize the JAR Location in Eclipse
Never use Add External JARs in Eclipse, as it writes absolute local file paths into the .classpath file that will break on every other machine and on your CI server.

Instead, enforce a project-relative lib/ folder:

Create a lib/ directory inside the project root.

Developers download the required JARs from Artifactory and drop them into lib/.

In Eclipse, right-click the project -> Build Path -> Configure Build Path -> Libraries -> Add JARs... (not External JARs) and select the files inside lib/.

Commit the .project, .classpath, and lib/ folder to your internal Git or SVN repository.

Now every developer and the build server share the exact same relative classpath.

2. Automate the Build on the CI Server
Once the JARs live in a predictable lib/ directory, you have three ways to automate the build in Jenkins, GitLab, or TeamCity:

Option A: Pair Eclipse with an Ant build.xml (Recommended)
Keep a simple build.xml in the root of the project that compiles src/ against lib/**/*.jar (just like the Ant example above).

Eclipse uses .classpath for coding, autocomplete, and local debugging.

The CI server runs ant package using the exact same lib/ folder.

Shortcut: Eclipse can actually generate this file for you via File -> Export -> General -> Ant Buildfiles. It reads your .classpath and writes a working build.xml, though you should clean out any hardcoded Eclipse install paths before committing it.

Option B: Use a Simple Build Script (javac + jar)
If you don't even want to install Ant on the build server, a 5-line Bash or PowerShell script in your CI pipeline can compile directly against the lib/ folder:

Bash
# Linux/macOS CI agent
mkdir -p build/classes
find src -name "*.java" > sources.txt
javac -cp "lib/*" -d build/classes @sources.txt
jar cf build/app.jar -C build/classes .
Option C: Headless Eclipse Build (Not Recommended)
You can install Eclipse on your CI server and run it from the command line without a UI (eclipse -nosplash -application org.eclipse.jdt.apt.core.aptBuild ...). In practice, companies avoid this because it is slow, fragile across Eclipse upgrades, and harder to debug than Ant or javac.

3. Avoid Bloating Git by Fetching from Artifactory in CI
Committing binary .jar files into Git works fine for small projects, but it quickly bloats repository size. Since you already have an internal Artifactory server, you can automate the manual download step without needing Maven:

Maintain a simple text file in your repo (e.g., dependencies.txt) listing the Artifactory URLs of the required JARs.

Add a pre-build step in your CI pipeline (and a helper script for developers) that downloads those JARs into lib/ using curl or the JFrog CLI (jf rt dl).

Add lib/*.jar to .gitignore.

Developers run the script once when they clone the repo (then refresh Eclipse), and the CI server runs the exact same script right before calling ant or javac.

How do I go about using gradle?

Switching to Gradle in an air-gapped environment with an internal Artifactory eliminates manual JAR downloads and keeps Eclipse and your CI server completely in sync.

1
Host the Gradle distribution on Artifactory
One-time setup by an admin
Normally, Gradle downloads itself from the internet. To avoid this, download the Gradle binary .zip (e.g., gradle-8.x-bin.zip) on an internet-connected machine, transfer it across the air-gap, and upload it to a generic repository in your internal Artifactory.

2
Configure the Gradle Wrapper
Generate the wrapper files in your project (using a local Gradle install via gradle wrapper), then edit gradle/wrapper/gradle-wrapper.properties so distributionUrl points to your internal Artifactory instead of services.gradle.org:

Properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https://artifactory.internal.company.com/artifactory/generic-local/gradle-8.x-bin.zip
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
Commit the gradlew, gradlew.bat, and gradle/ folder to Git. Now neither developers nor CI servers need Gradle pre-installed.

3
Point build.gradle to your internal Artifactory
Create a build.gradle file in the project root. Replace mavenCentral() with your internal Artifactory URL and list your dependencies:

Groovy
plugins {
    id 'java'
}

repositories {
    maven {
        url "https://artifactory.internal.company.com/artifactory/libs-release"
        // If your Artifactory requires login, add credentials here or in gradle.properties
    }
}

dependencies {
    // Pulled automatically from Artifactory
    implementation 'org.apache.commons:commons-lang3:3.14.0'
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.0'

    // Optional: keep using any manual JARs in lib/ while you migrate
    implementation fileTree(dir: 'lib', include: ['*.jar'])
}
4
Import the project into Eclipse
Modern versions of Eclipse IDE for Java Developers include the Buildship Gradle plugin out of the box (if yours doesn't, you will need to bring the Buildship update site zip across the air-gap).

Go to File - Import - Gradle - Existing Gradle Project.

Select your project root folder and click Finish.

Eclipse reads build.gradle, downloads the JARs from Artifactory into a local cache, and automatically configures the Build Path.

Whenever someone adds a new dependency to build.gradle, right-click the project and select Gradle - Refresh Gradle Project to update Eclipse's classpath.

5
Automate the build in CI
Point your internal Jenkins, GitLab, or TeamCity job to run the Gradle Wrapper from the repository root:

Bash
# Linux/macOS CI agent
./gradlew clean build

# Windows CI agent
gradlew.bat clean build
This compiles the code, runs your tests, and outputs the final .jar into build/libs/. If a build agent has no access to Artifactory at all (only a pre-populated local cache), append --offline to the command.

Tip for Artifactory authentication: Never hardcode Artifactory usernames or API tokens in build.gradle. Store them in each developer's local ~/.gradle/gradle.properties file and inject them as environment variables on your CI server.
