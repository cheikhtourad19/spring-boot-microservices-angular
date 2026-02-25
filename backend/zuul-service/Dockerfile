FROM maven:3.8.5-openjdk-11-slim AS builder

WORKDIR /build

COPY zuul-service ./zuul-service
RUN cd /build/zuul-service && mvn package -DskipTests -q

FROM eclipse-temurin:11-jre

WORKDIR /app

COPY --from=builder /build/zuul-service/target/zuul-service-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8090

ENTRYPOINT ["java", "-jar", "app.jar"]
