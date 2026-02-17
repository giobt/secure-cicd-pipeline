# =============================================================================
# STAGE 1: Builder - contains JDK, Maven, and build tools
# These tools are NOT carried into production (multi-stage build)
# =============================================================================
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /build

# Copy dependency descriptor first for better layer caching
COPY pom.xml .
RUN apk add --no-cache maven && \
    mvn dependency:go-offline -B

# Copy source code and build
COPY src ./src
RUN mvn package -DskipTests -B && \
    # Extract the layered JAR for optimal Docker layers
    java -Djarmode=layertools -jar target/*.jar extract --destination /extracted

# =============================================================================
# STAGE 2: Runtime - minimal image with only JRE and the application
# No build tools, no compilers, no package managers in production
# =============================================================================
FROM eclipse-temurin:21-jre-alpine AS runtime

# Run as non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy layers in order of least-to-most likely to change
COPY --from=builder /extracted/dependencies/ ./
COPY --from=builder /extracted/spring-boot-loader/ ./
COPY --from=builder /extracted/snapshot-dependencies/ ./
COPY --from=builder /extracted/application/ ./

# Set non-root user
USER appuser

EXPOSE 8080

HEALTHCHECK --interval=10s --timeout=3s --start-period=15s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]