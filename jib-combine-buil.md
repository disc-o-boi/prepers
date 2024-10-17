Using the **Jib Maven Plugin** to containerize and run your Angular app is a slightly different approach compared to a `Dockerfile`. Jib allows you to build container images directly from your Maven project without needing a `Dockerfile` or even having Docker installed locally.

However, **Jib** is primarily designed for containerizing Java applications. Since Angular is a front-end framework and typically produces static assets (like HTML, JavaScript, and CSS), you will need to follow an approach where **Jib** is used indirectly, by combining your Angular app with a backend server (like **Spring Boot**), which Jib can containerize.

Below are two scenarios depending on your setup:

1. **Angular app with no backend (served using NGINX)**: Use a Dockerfile (recommended in this case).
2. **Angular app bundled with a Spring Boot backend (using Jib)**: Use the Jib Maven plugin.

I'll focus on the second scenario where you have a **Spring Boot backend** and want to bundle the Angular app and then containerize the combined application using Jib.

### Step 1: Build Angular and Copy to Spring Boot Static Resources

First, configure your project to build your Angular app and copy the built static files to the Spring Boot `static` folder.

1. Build the Angular app with the following command:

    ```bash
    ng build --prod
    ```

    This will generate the production build of your Angular app inside the `dist/your-angular-app` folder.

2. Copy the contents of `dist/your-angular-app` to your Spring Boot application’s `src/main/resources/static/` folder.

This way, your Spring Boot application will serve the Angular files as static assets when the application runs.

### Step 2: Use Maven to Automate the Angular Build and Copy

To automate the process of building the Angular app and copying the build output to the Spring Boot resources folder during the Maven build process, you can add a **frontend-maven-plugin** to the Spring Boot `pom.xml`.

Here's how to modify the `pom.xml` of your Spring Boot project to include the steps for building Angular and copying the output:

```xml
<project>
    <!-- Spring Boot dependencies, plugins, etc. -->

    <build>
        <plugins>
            <!-- Jib Maven Plugin for building the container image -->
            <plugin>
                <groupId>com.google.cloud.tools</groupId>
                <artifactId>jib-maven-plugin</artifactId>
                <version>3.1.4</version>
                <configuration>
                    <!-- Name of the image to build -->
                    <to>
                        <image>your-dockerhub-username/my-spring-angular-app</image>
                    </to>
                    <!-- Expose necessary ports -->
                    <container>
                        <ports>
                            <port>8080</port>
                        </ports>
                    </container>
                </configuration>
            </plugin>

            <!-- Frontend Maven Plugin for building Angular -->
            <plugin>
                <groupId>com.github.eirslett</groupId>
                <artifactId>frontend-maven-plugin</artifactId>
                <version>1.12.0</version>

                <!-- Install Node.js and NPM -->
                <executions>
                    <execution>
                        <id>install-node-and-npm</id>
                        <goals>
                            <goal>install-node-and-npm</goal>
                        </goals>
                        <configuration>
                            <nodeVersion>v14.17.0</nodeVersion>
                            <npmVersion>6.14.13</npmVersion>
                        </configuration>
                    </execution>

                    <!-- Install NPM dependencies -->
                    <execution>
                        <id>npm-install</id>
                        <goals>
                            <goal>npm</goal>
                        </goals>
                        <configuration>
                            <arguments>install</arguments>
                        </configuration>
                    </execution>

                    <!-- Build the Angular project -->
                    <execution>
                        <id>npm-build</id>
                        <goals>
                            <goal>npm</goal>
                        </goals>
                        <configuration>
                            <arguments>run build -- --prod</arguments>
                        </configuration>
                    </execution>

                    <!-- Copy the Angular build output to the Spring Boot static folder -->
                    <execution>
                        <id>copy-angular-build</id>
                        <goals>
                            <goal>npm</goal>
                        </goals>
                        <phase>prepare-package</phase>
                        <configuration>
                            <workingDirectory>${project.basedir}/frontend</workingDirectory>
                            <environmentVariables>
                                <BUILD_DESTINATION>${project.basedir}/src/main/resources/static</BUILD_DESTINATION>
                            </environmentVariables>
                            <arguments>run-script copy-to-backend</arguments>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

### Explanation:

1. **Jib Plugin**: 
    - The Jib plugin is configured to build and push a Docker image to your Docker registry (like DockerHub).
    - It is configured to expose port `8080` (which is the default for Spring Boot).

2. **Frontend Maven Plugin**: 
    - This plugin installs Node.js and NPM, installs dependencies, and builds the Angular application.
    - It then copies the generated Angular build from `dist/your-angular-app` into the Spring Boot resources folder (`src/main/resources/static`), so Spring Boot can serve these static files.

### Step 3: Create the NPM Script to Copy Angular Build to Spring Boot

In the root of your Angular project (`frontend/` directory), modify the `package.json` to add a script to copy the Angular build files to the Spring Boot `static` folder:

```json
{
  "scripts": {
    "ng": "ng",
    "start": "ng serve",
    "build": "ng build",
    "test": "ng test",
    "lint": "ng lint",
    "e2e": "ng e2e",
    "copy-to-backend": "cp -r dist/your-angular-app/* $BUILD_DESTINATION"
  }
}
```

This script (`copy-to-backend`) will copy the Angular build files from the `dist` folder to the destination directory defined by the Maven configuration.

### Step 4: Build the Container with Jib

Now that everything is configured, you can build the Docker image using the **Jib Maven Plugin**. Since Jib doesn't require Docker to be installed locally, it simplifies the containerization process.

Run the following Maven command to build and push your image:

```bash
mvn clean install jib:build
```

This will:

1. Build the Angular project.
2. Copy the Angular build to the Spring Boot static folder.
3. Containerize the combined Spring Boot + Angular application using Jib.
4. Push the Docker image to your Docker registry (such as DockerHub).

### Step 5: Run the Container

Once the image is built and pushed to your Docker registry, you can pull and run the container using Docker:

```bash
docker run -p 8080:8080 your-dockerhub-username/my-spring-angular-app
```

This command maps port `8080` on your local machine to port `8080` in the container, allowing you to access the application at `http://localhost:8080`.

### Conclusion

By using the **Jib Maven Plugin**, you can easily containerize and deploy a combined Spring Boot and Angular application without needing a `Dockerfile`. Jib simplifies the build process, automatically pushing the image to your Docker registry.

If you want to serve a standalone Angular application using **NGINX** or another server, it's generally easier to stick with a `Dockerfile` approach, but for a Spring Boot + Angular project, Jib provides a smooth and integrated workflow.


#############

The error **"Main class was not found"** usually occurs when **Jib** cannot locate the main class in your Java application during the containerization process. This is typically a configuration issue with Jib or Maven, or it could be caused by the structure of your project.

Below are some steps you can follow to resolve this issue.

### Step 1: Check `pom.xml` for the Main Class

Ensure that your `pom.xml` is correctly configured, particularly if you're using a **Spring Boot** application. The main class needs to be properly specified so that Jib can find it and build the container correctly.

In your `pom.xml`, you may need to explicitly define the main class under the Jib plugin's configuration, like so:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>com.google.cloud.tools</groupId>
            <artifactId>jib-maven-plugin</artifactId>
            <version>3.1.4</version>
            <configuration>
                <!-- Specify the fully qualified name of the main class -->
                <mainClass>com.example.yourpackage.YourApplicationClass</mainClass>
            </configuration>
        </plugin>
    </plugins>
</build>
```

Replace `com.example.yourpackage.YourApplicationClass` with the fully qualified name of the main class (i.e., the class with the `public static void main(String[] args)` method).

For **Spring Boot**, this is typically the class annotated with `@SpringBootApplication`, for example:

```java
@SpringBootApplication
public class YourApplicationClass {
    public static void main(String[] args) {
        SpringApplication.run(YourApplicationClass.class, args);
    }
}
```

### Step 2: Ensure Your Project is Packaged as a JAR

For a Java project, ensure that it is packaged correctly as a **JAR** file with a manifest that correctly identifies the main class. To check that, ensure that your **Maven `pom.xml`** has the following in its build section:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

This plugin helps ensure that the **Spring Boot** application is built correctly, including the manifest file with the correct `Main-Class` entry.

You can verify this by running:

```bash
mvn clean package
```

Then inspect the `META-INF/MANIFEST.MF` file within the resulting `jar` file (you can unzip the jar file) to check if the `Main-Class` is specified correctly:

```bash
Main-Class: com.example.yourpackage.YourApplicationClass
```

### Step 3: Ensure Proper Project Structure

Make sure that your Java source code is located in the correct directory structure, as required by Maven. Maven expects the main Java files to be in:

```
src/main/java/com/example/yourpackage/YourApplicationClass.java
```

If your classes are not in the correct directory, Jib and Maven may not find the main class.

### Step 4: Ensure Dependencies are Correct

Sometimes this issue can arise if there are missing dependencies or incompatible versions of libraries. For example, if you're using Spring Boot, ensure that you have the correct dependencies for Spring Boot, as well as other dependencies your project might need.

In your `pom.xml`, ensure you have something like this for Spring Boot:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- Other dependencies -->
</dependencies>
```

If there are issues with dependencies, Jib might fail to identify the main class properly.

### Step 5: Clean and Rebuild the Project

Sometimes, Maven's build cache or target directory may become stale, and a clean build might help. Run the following commands to clean and rebuild the project:

```bash
mvn clean install
mvn jib:build
```

This will clean up any old files and ensure everything is built from scratch.

### Step 6: Double Check Your Maven `pom.xml` Jib Configuration

Ensure that your `pom.xml` does not have conflicting configurations. Below is an example of a properly configured **Jib** plugin for a **Spring Boot** application:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>com.google.cloud.tools</groupId>
            <artifactId>jib-maven-plugin</artifactId>
            <version>3.1.4</version>
            <configuration>
                <to>
                    <image>mycontainerregistry.azurecr.io/my-app</image>
                </to>
                <!-- Fully qualified name of the main class -->
                <mainClass>com.example.yourpackage.YourApplicationClass</mainClass>
                <!-- Configure ports -->
                <container>
                    <ports>
                        <port>8080</port>
                    </ports>
                </container>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### Step 7: Jib `jar` or `war` Packaging

Make sure your **packaging** type is correctly set in your `pom.xml`. If you're building a **Spring Boot** application that is packaged as a `jar`, your `pom.xml` should look like this:

```xml
<packaging>jar</packaging>
```

If you're building a web application that uses **WAR packaging**, ensure this is properly set:

```xml
<packaging>war</packaging>
```

Jib will behave differently based on whether it's packaging a `jar` or `war`, so it's important to specify this correctly.

### Step 8: Use Jib's `jar` Mode (Optional)

If you still face issues, you can explicitly tell Jib to use the **Jar processing mode** in the `pom.xml`:

```xml
<configuration>
    <containerizingMode>packaged</containerizingMode>
</configuration>
```

This instructs Jib to containerize the pre-built JAR file, which may help if Jib is failing to locate the main class in its default mode.

---

### Recap

If you're seeing the error **"Main class was not found"** when using Jib, follow these steps:

1. **Add the `mainClass` to Jib's configuration** in `pom.xml`.
2. **Ensure the project's directory structure** follows Maven conventions.
3. **Check your `META-INF/MANIFEST.MF`** file to verify that the main class is correctly specified.
4. **Clean and rebuild the project** to ensure that the build artifacts are up to date.
5. **Ensure dependencies and Spring Boot setup** in your `pom.xml` are correct.

Once these are set up correctly, running `mvn jib:build` should successfully package your Java app into a container and push it to Azure Container Registry (or another registry).