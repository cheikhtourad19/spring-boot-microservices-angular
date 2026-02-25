FROM maven:3.8.5-openjdk-11-slim AS builder

WORKDIR /build

COPY pom.xml ./pom.xml
RUN mvn -f /build/pom.xml install -N -DskipTests -q

COPY common-service ./common-service
COPY common-exam ./common-exam
COPY common-student ./common-student

RUN mvn -f /build/common-service/pom.xml install -DskipTests -q
RUN mvn -f /build/common-exam/pom.xml install -DskipTests -q
RUN mvn -f /build/common-student/pom.xml install -DskipTests -q

COPY answer-service ./answer-service
RUN mvn -f /build/answer-service/pom.xml package -DskipTests -q

FROM eclipse-temurin:11-jre

WORKDIR /app

COPY --from=builder /build/answer-service/target/answer-service-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8083

ENTRYPOINT ["java", "-jar", "app.jar"]
