FROM maven:3.8.5-openjdk-11-slim AS builder

WORKDIR /build

COPY pom.xml ./pom.xml
RUN mvn -f /build/pom.xml install -N -DskipTests -q

COPY zuul-service ./zuul-service
RUN mvn -f /build/zuul-service/pom.xml package -DskipTests -q

FROM eclipse-temurin:11-jre

WORKDIR /app

COPY --from=builder /build/zuul-service/target/zuul-service-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8091

ENTRYPOINT ["java", "-jar", "app.jar"]
