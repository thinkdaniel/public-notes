# Deploying Spring Boot Applications to Heroku with CircleCI

## Overview

In this lesson, you'll learn how to set up a complete CI/CD pipeline using CircleCI to automatically test, build, and deploy your Spring Boot application to Heroku with database integration.

## Prerequisites

Before starting this lesson, ensure you have:
- A Spring Boot application with Maven
- A GitHub repository containing your Spring Boot project
- A CircleCI account (free tier available)
- A Heroku account
- Docker Hub account
- Basic understanding of Spring Boot and Maven

## What You'll Learn

By the end of this lesson, you'll be able to:
- Configure CircleCI to automatically test your Spring Boot application
- Build and publish Docker images using CircleCI
- Deploy containerized applications to Heroku
- Configure database connections for Heroku PostgreSQL
- Set up automated deployment pipelines triggered by Git commits

## Part 1: Understanding the CI/CD Pipeline

Our pipeline consists of four main stages:

1. **Test**: Run unit tests to ensure code quality
2. **Build**: Create a Docker image of your application
3. **Publish**: Push the Docker image to Docker Hub
4. **Deploy**: Deploy the containerized application to Heroku

## Part 2: Setting Up Your CircleCI Configuration

Create a `.circleci/config.yml` file in your project root:

```yaml
version: 2.1

orbs:
  docker: circleci/docker@3.0.0
  heroku: circleci/heroku@2.0.0

jobs:
  test:
    docker:
      - image: cimg/openjdk:21.0
    steps:
      - checkout
      - restore_cache:
          key: maven-repo-{{ checksum "pom.xml" }}
      - run:
          name: Download dependencies
          command: ./mvnw dependency:go-offline -B -ntp
      - save_cache:
          key: maven-repo-{{ checksum "pom.xml" }}
          paths:
            - ~/.m2
      - run:
          name: Run tests
          command: ./mvnw test -Dspring.profiles.active=test

  build:
    executor: docker/docker
    steps:
      - setup_remote_docker:
          docker_layer_caching: true
      - checkout
      - docker/build:
          image: your-dockerhub-username/your-app-name
          tag: << pipeline.git.revision >>
      - run:
          name: Save Docker image to workspace
          command: |
            mkdir -p /tmp/docker-images
            docker save your-dockerhub-username/your-app-name:<< pipeline.git.revision >> -o /tmp/docker-images/your-app.tar
      - persist_to_workspace:
          root: /tmp/docker-images
          paths:
            - "*.tar"

  publish:
    executor: docker/docker
    steps:
      - setup_remote_docker
      - attach_workspace:
          at: /tmp/docker-images
      - run:
          name: Load Docker image from workspace and tag it
          command: |
            docker load -i /tmp/docker-images/your-app.tar
            docker tag your-dockerhub-username/your-app-name:<< pipeline.git.revision >> your-dockerhub-username/your-app-name:latest
      - docker/check
      - docker/push:
          image: your-dockerhub-username/your-app-name
          tag: << pipeline.git.revision >>,latest

  deploy:
    executor: docker/docker
    steps:
      - setup_remote_docker
      - heroku/install
      - run:
          name: Set Heroku stack to container
          command: heroku stack:set container -a $HEROKU_APP_NAME
      - run:
          name: Login to Heroku
          command: heroku container:login
      - run:
          name: Transform DATABASE_URL for Spring Boot
          command: |
            # Get Heroku DATABASE_URL and parse components
            DATABASE_URL=$(heroku config:get DATABASE_URL -a $HEROKU_APP_NAME)
            echo "Original DATABASE_URL format detected"
            
            # Extract database components from postgres://user:pass@host:port/db
            DB_USER=$(echo $DATABASE_URL | sed 's/postgres:\/\/\([^:]*\):.*/\1/')
            DB_PASS=$(echo $DATABASE_URL | sed 's/postgres:\/\/[^:]*:\([^@]*\)@.*/\1/')
            DB_HOST=$(echo $DATABASE_URL | sed 's/.*@\([^:]*\):.*/\1/')
            DB_PORT=$(echo $DATABASE_URL | sed 's/.*:\([0-9]*\)\/.*/\1/')
            DB_NAME=$(echo $DATABASE_URL | sed 's/.*\/\(.*\)/\1/')
            
            # Create clean JDBC URL without embedded credentials
            JDBC_URL="jdbc:postgresql://$DB_HOST:$DB_PORT/$DB_NAME"
            
            # Set Spring Boot environment variables
            heroku config:set \
              JDBC_DATABASE_URL="$JDBC_URL" \
              DATABASE_USERNAME="$DB_USER" \
              DATABASE_PASSWORD="$DB_PASS" \
              DATABASE_SSL=true \
              DATABASE_SSL_MODE=require \
              -a $HEROKU_APP_NAME
            
            echo "Database configuration updated for Spring Boot"
      - run:
          name: Pull image from Docker Hub and retag for Heroku
          command: |
            docker pull your-dockerhub-username/your-app-name:<< pipeline.git.revision >>
            docker tag your-dockerhub-username/your-app-name:<< pipeline.git.revision >> registry.heroku.com/$HEROKU_APP_NAME/web
            docker push registry.heroku.com/$HEROKU_APP_NAME/web
      - heroku/release-docker-image:
          app-name: $HEROKU_APP_NAME
          process-types: web

workflows:
  build-test-deploy:
    jobs:
      - test
      - build:
          requires:
            - test
      - publish:
          requires:
            - build
          filters:
            branches:
              only: main
      - deploy:
          requires:
            - publish
          filters:
            branches:
              only: main
```

## Part 3: Understanding Each Job

### Test Job
- Uses OpenJDK 21 container
- Caches Maven dependencies for faster builds
- Runs tests with a test profile (`-Dspring.profiles.active=test`)
- Uses H2 in-memory database for integration tests
- Prevents broken code from being deployed

**Important**: The test job runs in a containerized environment without access to external databases. Your Spring Boot application must be configured to use H2 in-memory database for the test profile, otherwise integration tests that require database access will fail.

### Build Job
- Uses Docker executor with layer caching
- Builds Docker image tagged with Git revision
- Saves image to workspace for next job
- Enables efficient Docker builds

### Publish Job
- Loads Docker image from workspace
- Tags image as both revision-specific and latest
- Pushes to Docker Hub registry
- Only runs for main branch commits

### Deploy Job
- Configures Heroku for container deployment
- Transforms Heroku DATABASE_URL for Spring Boot compatibility
- Pulls image from Docker Hub and pushes to Heroku registry
- Releases the new version to production

## Part 4: Spring Boot Database Configuration

Update your `application.yml` to work with the environment variables and ensure proper test configuration:

```yaml
spring:
  datasource:
    url: ${JDBC_DATABASE_URL:jdbc:h2:mem:testdb}
    username: ${DATABASE_USERNAME:sa}
    password: ${DATABASE_PASSWORD:password}
    driver-class-name: ${DATABASE_DRIVER:org.h2.Driver}
  
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false
    
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:development}

---
spring:
  config:
    activate:
      on-profile: test
  datasource:
    url: jdbc:h2:mem:testdb
    username: sa
    password: password
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: false
    database-platform: org.hibernate.dialect.H2Dialect

---
spring:
  config:
    activate:
      on-profile: production
  datasource:
    driver-class-name: org.postgresql.Driver
  jpa:
    database-platform: org.hibernate.dialect.PostgreSQLDialect
    hibernate:
      ddl-auto: validate
```

### Critical Test Configuration Notes

**For Integration Tests**: Your integration tests that interact with the database must work with the H2 in-memory database configured in the test profile. The CircleCI test job runs in an isolated container without access to external databases like PostgreSQL.

**Best Practices for Test Database Setup**:
- Use `create-drop` for test profile to ensure clean state between test runs
- Keep test data lightweight and focused on testing business logic
- Consider using `@Sql` annotations to populate test data
- Use `@Transactional` with `@Rollback` for test isolation

## Part 5: Required Setup Steps

### 1. Ensure H2 Dependency for Tests
Add H2 dependency to your `pom.xml` if not already present:

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

### 2. Create Dockerfile
Add a `Dockerfile` to your project root:

```dockerfile
FROM openjdk:21-jre-slim
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### 2. Set Up CircleCI Environment Variables
In your CircleCI project settings, add:
- `DOCKER_LOGIN`: Your Docker Hub username
- `DOCKER_PASSWORD`: Your Docker Hub password
- `HEROKU_API_KEY`: Your Heroku API key
- `HEROKU_APP_NAME`: Your Heroku application name

### 3. Create Heroku App
```bash
# Install Heroku CLI and login
heroku create your-heroku-app-name

# Add PostgreSQL database
heroku addons:create heroku-postgresql:essential-0 -a your-heroku-app-name
```

### 4. Update Configuration Placeholders
Replace these placeholders in your config:
- `your-dockerhub-username`: Your Docker Hub username
- `your-app-name`: Your application name

**Note**: You no longer need to replace `your-heroku-app-name` throughout the config since we're using the `HEROKU_APP_NAME` environment variable.

## Benefits of Using Environment Variables

Using `HEROKU_APP_NAME` as an environment variable provides several advantages:

- **Reusability**: The same configuration can be used across multiple projects by simply changing the environment variable
- **Security**: App names are centrally managed in CircleCI's encrypted environment variables
- **Maintainability**: Easy to update app names without modifying the configuration file
- **Multi-environment support**: Different branches can deploy to different Heroku apps by using different environment variable values

## Part 6: Database URL Transformation Explained

Heroku provides database connection info as: `postgres://user:pass@host:port/db`

Spring Boot expects separate configuration properties, so the deploy job transforms this by:
1. Parsing the Heroku DATABASE_URL
2. Extracting individual components (user, password, host, port, database)
3. Creating a clean JDBC URL
4. Setting individual environment variables for Spring Boot

## Part 7: Workflow Logic

The workflow ensures:
- Tests must pass before building
- Images are only published from the main branch
- Deployment only happens after successful publishing
- Failed tests prevent deployment

## Part 8: Testing Your Setup

1. Commit your configuration files to your repository
2. Push to the main branch
3. Watch the CircleCI pipeline execute
4. Verify your application is deployed to Heroku
5. Test database connectivity

## Common Troubleshooting

### Database Connection Issues
- Ensure PostgreSQL driver is in your `pom.xml`
- Check that environment variables are set correctly in Heroku
- Verify SSL configuration for Heroku PostgreSQL

### Test Failures in CircleCI
- **Integration test failures**: Ensure your tests use H2 database with the test profile
- **Database dependency issues**: Verify H2 is available as a test dependency
- **Profile configuration**: Check that `spring.profiles.active=test` is properly configured
- **Test isolation**: Use `@Transactional` and `@Rollback` for database test isolation

### Docker Build Failures
- Check that your Dockerfile is in the project root
- Ensure Maven builds successfully locally
- Verify Docker Hub credentials in CircleCI

### Heroku Deployment Issues
- Confirm Heroku API key is set correctly
- Check that your app name matches across all configurations
- Ensure container stack is set on your Heroku app

## Next Steps

Consider enhancing your pipeline with:
- Integration tests against a test database
- Security scanning of Docker images
- Blue-green deployments
- Database migrations using Flyway or Liquibase
- Monitoring and alerting setup

## Summary

You now have a complete CI/CD pipeline that automatically tests, builds, and deploys your Spring Boot application to Heroku with proper database integration. This setup ensures code quality, enables rapid deployment, and provides a solid foundation for production applications.