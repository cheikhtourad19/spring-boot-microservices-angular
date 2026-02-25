FROM maven:3.8.5-openjdk-11-slim AS builder

WORKDIR /build

COPY pom.xml ./pom.xml
RUN mvn install -N -DskipTests -q

COPY eureka-service ./eureka-service
RUN cd /build/eureka-service && mvn package -DskipTests -q

FROM eclipse-temurin:11-jre

WORKDIR /app

COPY --from=builder /build/eureka-service/target/eureka-service-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8761

ENTRYPOINT ["java", "-jar", "app.jar"]
