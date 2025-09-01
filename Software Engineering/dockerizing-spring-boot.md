# Dockerizing a Spring Boot Application

This lesson explains how to create a Docker image for a Spring Boot application, allowing it to run in a containerized environment.

## Why Dockerize a Spring Boot App

A Spring Boot app could be deployed without Docker, but containerizing it offers several advantages:

1. **Simplified Deployment**: Docker images encapsulate all dependencies, making it easier to deploy the application across different environments without worrying about missing libraries or configuration issues.

1. **Environment Consistency**: Docker ensures that the application runs in the same environment regardless of where it is deployed, reducing the "it works on my machine" problem.

1. **Scalability**: Docker makes it easy to scale the application by running multiple instances of the container, allowing for better resource utilization and load balancing.

1. **Isolation**: Each container runs in its own environment, preventing conflicts between different applications or services running on the same host.

1. **Easier Rollbacks**: With Docker, you can easily roll back to a previous version of the application by deploying an older image, simplifying the process of managing application updates.

Typically, the dockerized Spring Boot app is deployed to staging or production environments.

In development environment, we do not typically use Docker because we would lose the hot-reloading feature provided by Spring Boot DevTools, which significantly speeds up the development process. Additionally, debugging inside a Docker container can be more complex and less efficient than running the application directly on the host machine. Therefore, for local development, it's often more practical to run the Spring Boot application directly on the host system rather than within a Docker container.

## Dockerfile and .dockerignore

The Dockerfile is a text file that contains all the commands to assemble an image. It defines the environment in which your application will run.

By convention, the Dockerfile is named `Dockerfile` (without any file extension) and is located in the root directory of your project. A custom name could be used by specifying the `-f` option with the `docker build` command.

A `.dockerignore` file to exclude files and directories from the build context. This file works similarly to a `.gitignore` file and helps to reduce the size of the build context sent to the Docker daemon.

For our Spring Boot project, a sample `.dockerignore` file is here.

We will use a multi-stage Docker build to create a lightweight image for our Spring Boot application. This approach allows us to separate the build environment from the runtime environment, resulting in a smaller final image.

### Build Stage

The build stage is responsible for compiling the application and creating the JAR file.

```dockerfile
FROM eclipse-temurin:21-jdk AS build

# Set the working directory
WORKDIR /app

# Copy Maven wrapper, .mvn directory, and pom.xml first
# This is to maximize layer caching
COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .

# Download dependencies (so they are cached unless pom.xml changes)
# Maven will download dependencies into local repo (~/.m2/repository)
# -B flag is used for batch mode, which disables interactive prompts
RUN ./mvnw dependency:go-offline -B

# Copy the rest of the application source code
COPY . .

# Build the app
RUN ./mvnw clean package -DskipTests -ntp
```

First, we use the Eclipse Temurin JDK 21 image as the base image for the build stage. This image includes the JDK and is optimized for running Java applications.

Next, we set the working directory to `/app` using the `WORKDIR` instruction. This is where our application code will reside inside the container.

After that we copy the Maven wrapper `mvnw`, the `.mvn` folder, and `pom.xml` file into the working directory. This allows us to take advantage of Docker's layer caching by only re-downloading dependencies if the `pom.xml` file changes. Recall that Docker uses layers - this means each instruction in the Dockerfile creates a new layer in the image. If a layer hasn't changed, Docker can use the cached version instead of rebuilding it, which speeds up the build process.

We then install the application dependencies using the `./mvnw dependency:go-offline -B` command. This downloads all the required dependencies and caches them in the local Maven repository, so they don't need to be downloaded again in subsequent builds. The `-B` flag is used for batch mode, which disables interactive prompts. This is useful in a CI/CD environment where you want the build to run without any manual intervention.

Following that, we can copy the rest of the application source code into the container.

The app is then built using the `./mvnw clean package -DskipTests` command, which compiles the code and packages it into a JAR file. The `-DskipTests` flag is used to skip the tests during the build process, which can speed up the build time, especially in a CI/CD environment. The `-ntp` flag is used to disable the transfer progress logging, which can help reduce the amount of output generated during the build process.

### Runtime Stage

In the runtime stage, we use the Eclipse Temurin JRE 21 image as the base image as we no longer need the full JDK to run the application. This JRE image is smaller in size compared to the earlier JDK image, which helps reduce the overall size of the Docker image.

The output is a minimal image that only contains the JAR file and the necessary runtime dependencies.

```dockerfile
FROM eclipse-temurin:21-jre AS runtime

# Set the working directory
WORKDIR /app

# Copy the JAR file from the build stage
COPY --from=build /app/target/*.jar app.jar

# Create non-root user for security
RUN groupadd -r spring && useradd -r -g spring spring

# Change ownership to spring user
RUN chown spring:spring app.jar

# Switch to non-root user
USER spring

# Run the application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

We set the working directory to `/app` and copy the JAR file from the build stage into the runtime image.

For security reasons, we create a non-root user and switch to that user before running the application. This minimizes the risk of privilege escalation attacks and follows best practices for running applications in containers.

Finally, we specify the command to run the application using the `ENTRYPOINT` instruction.

### Building the Docker Image

To build the Docker image, run the following command in the root directory of your project:

```bash
docker build -t my-spring-boot-app .
```

This command builds the Docker image and tags it as `my-spring-boot-app`.

You can check the image is built successfully by running `docker images` or `docker image ls`.

### Running the Docker Container

To run the Docker container, use the following command:

```bash
docker run -p 8080:8080 my-spring-boot-app
```

This command runs the container and maps port 8080 on the host to port 8080 in the container, allowing you to access the application at `http://localhost:8080`.

#### Note on localhost Database

When running the application in a Docker container, the database connection settings may need to be adjusted. If the application is configured to connect to a database running on `localhost`, `localhost` now refers to the container itself, not the host machine. To connect to a database running on the host, you may need to use the host's IP address or a special DNS name like `host.docker.internal`.

For this reason, it is recommended to externalize the database connection settings (e.g., using environment variables) so that they can be easily configured for different environments. For example, in your `application.properties`.

```
spring.datasource.url=${DATABASE_URL:jdbc:postgresql://localhost:5432/mydb}
spring.datasource.username=${DATABASE_USERNAME:myuser}
spring.datasource.password=${DATABASE_PASSWORD:mypass}
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
```

When you start the container, you can pass the database connection settings as environment variables.

```bash
docker run \
  -e DATABASE_URL=jdbc:postgresql://host.docker.internal:5432/mydb \
  -e DATABASE_USERNAME=myuser \
  -e DATABASE_PASSWORD=mypass \
  -p 8080:8080 \
  my-spring-boot-app
```

### Conclusion

In this guide, we covered the process of dockerizing a Spring Boot application using a multi-stage Docker build. This approach helps to create a lightweight and efficient Docker image that can be easily deployed in various environments.
