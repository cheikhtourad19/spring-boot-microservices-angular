FROM maven:3.8.5-openjdk-11-slim AS builder

WORKDIR /build

COPY pom.xml ./pom.xml
RUN mvn install -N -DskipTests -q

COPY common-service ./common-service
COPY common-student ./common-student

RUN cd /build/common-service && mvn install -DskipTests -q
RUN cd /build/common-student && mvn install -DskipTests -q

COPY user-service ./user-service
RUN cd /build/user-service && mvn package -DskipTests -q

FROM eclipse-temurin:11-jre

WORKDIR /app

COPY --from=builder /build/user-service/target/user-service-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
