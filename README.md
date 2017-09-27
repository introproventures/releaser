
Nexus Releaser
--------

### Summary

It's a minimal a reference example for setting up your project to deploy to Maven Central with staging release to Nexus Sonatype repository https://oss.sonatype.org/

This project is based on these instructions from: https://github.com/davidcarboni/releaser

### Using Nexus Releaser 

There are two files of interest in this project:
 * `pom.xml` - a POM file that meets Maven Central requirements and defines a deployment configuration
 * `settings.xml` - a Maven user settings file that defines release deployment configuration

These are set up initially to test with a local Nexus instance, but also contain the Sonatype OSS repository information, commented out.

You'll also need to follow the instructions for setting up a Sonatype Jira account and creating a ticket here: http://central.sonatype.org/pages/ossrh-guide.html#create-a-ticket-with-sonatype

### GPG key generation

Here's a terse summary of the GPG commands you'll need to generate and publish a key for signing your artifacts. For the full version, see https://docs.sonatype.org/display/Repository/How+To+Generate+PGP+Signatures+With+Maven

    # is GPG installed?
    gpg --version 
    
    # Generate a key. If in doubt, accept defaluts. I'd recommend a 3072-bit key:
    gpg --gen-key
    
    # Check what was generated:
    gpg --list-keys
    gpg --list-secret-keys
    # You should see:
    # pub nnnnA/FFFFFFFF YY-MM-DD
    # ... etc ...
    # make a note of FFFFFFFF - this is your key ID
    
    # Check for sub keys:
    gpg --edit-key FFFFFFFF
    # If any "sub" key says "usage: S", refer to the URL above for instructions on deleting it.
    
    # Publish key:
    gpg --keyserver hkp://pool.sks-keyservers.net --send-keys FFFFFFFF


### Test with a local Nexus instance

Try running with a local Nexus instance to get the hang of it:
 * Map docker machine host port 8081 to guest container port 8081 in VirtualBox machine instance network settings.
 * Run Nexus Docker image with binding the exposed port 8081 to the host: `docker run -d -p 8081:8081 --name nexus sonatype/nexus:oss`
 * Test Nexus instance is running: `curl http://localhost:8081/nexus/service/local/status`
 * Check instance logs with `docker logs nexus`
 * open [http://localhost:8081/nexus](http://localhost:8081/nexus "If you have Nexus installed and running locally this link will work for you")
 * the default Nexus deployment username and password are already set in the example Maven `settings.xml` in the root of this project.
 * if you do practice releases, note that the tags created by the release plugin will get pushed to your remote so you'll need to delete them after practice runs. Deleting remote tags isn't obvious so here's a post that explains how: [http://nathanhoad.net/how-to-delete-a-remote-git-tag](http://nathanhoad.net/how-to-delete-a-remote-git-tag). 

The summary is:
 
     git tag -d 0.0.1
     git push origin :refs/tags/0.0.1

### Deploy artifact locally

Run `mvn clean deploy` to build and deploy your project in local Maven repository

### Maven Nexus release POM configuration

The standard Maven plugin used by a Release Process is the maven-release-plugin – the configuration for this plugin is minimal:
```
<plugin>
   <groupId>org.apache.maven.plugins</groupId>
   <artifactId>maven-release-plugin</artifactId>
   <version>2.4.2</version>
   <configuration>
      <tagNameFormat>@{project.version}</tagNameFormat>
      <autoVersionSubmodules>true</autoVersionSubmodules>
      <releaseProfiles>releases</releaseProfiles>
   </configuration>
</plugin>
```

What is important here is that the releaseProfiles configuration will actually force a Maven profile – the releases profile – to become active during the Release process.

It is in this process that the nexus-staging-maven-plugin is used to perform a deploy to the localhost-nexus-staging Nexus repository:
```
<profiles>
   <profile>
      <id>releases</id>
      <build>
         <plugins>
            <plugin>
               <groupId>org.sonatype.plugins</groupId>
               <artifactId>nexus-staging-maven-plugin</artifactId>
               <version>1.6.7</version>
               <executions>
                  <execution>
                     <id>default-deploy</id>
                     <phase>deploy</phase>
                     <goals>
                        <goal>deploy</goal>
                     </goals>
                  </execution>
               </executions>
               <configuration>
                  <serverId>localhost-nexus-staging</serverId>
                  <nexusUrl>http://localhost:8081/nexus/</nexusUrl>
                  <skipStaging>true</skipStaging>
               </configuration>
            </plugin>
         </plugins>
      </build>
   </profile>
</profiles>
```
The plugin is initially configured to perform the Release process without the staging mechanism (skipStaging=true).

If you perform release to a local Nexus OSS repository, you will need to skip staging with `<skipStaging>true</skipStaging>` tag in nexus-staging-maven-plugin `<configuration>`. Otherwise, you will get an error `Nexus instance  does not support staging workflow v2`, i.e.:

```
    <configuration>
        <serverId>localhost-nexus-staging</serverId>
        <nexusUrl>http://localhost:8081/nexus</nexusUrl>
        <skipStaging>true</skipStaging>
    </configuration>
```

For public Nexus releases, use the following configuration in `nexus-staging-maven-plugin`:
```
    <configuration>
        <serverId>sonatype-nexus-staging</serverId>
        <nexusUrl>https://oss.sonatype.org/</nexusUrl>
        <autoReleaseAfterClose>false</autoReleaseAfterClose>
    </configuration>
```

### Maven release process to Nexus repository

Let’s break apart the release process into small and focused steps. We are performing a Release when the current version of the project is a SNAPSHOT version – say 0.0.1-SNAPSHOT.
 
Ensure your artifact version is a -SNAPSHOT, then run the following:
 
1. `mvn release:clean`

Cleaning a Release will:

delete the release descriptor (release.properties)
delete any backup POM files

2. `mvn release:prepare`

Next part of the Release process is Preparing the Release; this will:

perform some checks – there should be no uncommitted changes and the project should depend on no SNAPSHOT dependencies
change the version of the project in the pom file to a full release number (remove SNAPSHOT suffix) – in our example – 0.0.1
run the project test suites
commit and push the changes
create the tag out of this non-SNAPSHOT versioned code
increase the version of the project in the pom – in our example – 0.0.2-SNAPSHOT
commit and push the changes

3. `mvn release:perform`

The last part of the Release process is Performing the Release; this will:

checkout release tag from SCM
build and deploy released code
auto-sign artifacts with your `gpg.passphrase` from `settings.xml`
This step of the process relies on the output of the Prepare step – the release.properties.

For more options, including rolling back a release or cleaning up a failed release, see: http://maven.apache.org/maven-release/maven-release-plugin/

If you find that the release is deploying a SNAPSHOT version instead of the release artifact, it may be because the release process has changed since I first uploaded this project to Github. I've updated these instructions to match, so just check your release plugin configuration matches the example in this project.

### Next steps

Once you're set up with Sonatype, have deployed your artifacts and commented on your Jira ticket, you'll need to follow these instructions in order to complete the process:

http://central.sonatype.org/pages/ossrh-guide.html

Here are the specific Maven instructions, including the `nexus-staging-maven-plugin` setup:

http://central.sonatype.org/pages/apache-maven.html

### Credits

David Carboni [https://github.com/davidcarboni/releaser](https://github.com/davidcarboni/releaser)

Baeldung [http://www.baeldung.com/maven-release-nexus](http://www.baeldung.com/maven-release-nexus)
