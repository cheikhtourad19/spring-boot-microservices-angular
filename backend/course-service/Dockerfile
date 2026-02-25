FROM maven:3.8.5-openjdk-11-slim AS builder

WORKDIR /build

COPY common-service ./common-service
COPY common-exam ./common-exam
COPY common-student ./common-student

RUN cd /build/common-service && mvn install -DskipTests -q
RUN cd /build/common-exam && mvn install -DskipTests -q
RUN cd /build/common-student && mvn install -DskipTests -q

COPY course-service ./course-service
RUN cd /build/course-service && mvn package -DskipTests -q

FROM eclipse-temurin:11-jre

WORKDIR /app

COPY --from=builder /build/course-service/target/course-service-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8081

ENTRYPOINT ["java", "-jar", "app.jar"]
