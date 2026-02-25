FROM maven:3.8.5-openjdk-11-slim AS builder

WORKDIR /build

# Copy and install common modules (paths relative to backend/ build context)
COPY common-service ./common-service
COPY common-exam ./common-exam

RUN cd /build/common-service && mvn install -DskipTests -q
RUN cd /build/common-exam && mvn install -DskipTests -q

# Copy and build exam-service
COPY exam-service ./exam-service
RUN cd /build/exam-service && mvn package -DskipTests -q

# ---- Runtime ----
FROM eclipse-temurin:11-jre

WORKDIR /app

COPY --from=builder /build/exam-service/target/examen-service-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8082

ENTRYPOINT ["java", "-jar", "app.jar"]
