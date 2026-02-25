FROM maven:3.8.5-openjdk-11-slim AS builder

WORKDIR /build

COPY pom.xml ./pom.xml
RUN mvn install -N -DskipTests -q

COPY api-gateway-service ./api-gateway-service
RUN cd /build/api-gateway-service && mvn package -DskipTests -q

FROM eclipse-temurin:11-jre

WORKDIR /app

COPY --from=builder /build/api-gateway-service/target/api-gateway-service-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8090

ENTRYPOINT ["java", "-jar", "app.jar"]
