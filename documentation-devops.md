# Documentation DevOps — Projet Spring Boot Microservices Angular

> **Instructions pour l'agent LaTeX :**
> - Convertir ce document en LaTeX complet (classe `report` ou `article`).
> - Utiliser le package `listings` ou `minted` pour tous les blocs de code.
> - Utiliser le package `graphicx` pour toutes les figures.
> - Pour chaque image, générer une figure LaTeX avec `\includegraphics[width=\linewidth]{images/placeholder.png}` **remplacé par le nom de fichier indiqué dans la légende**, et laisser la source vide (`{}`), accompagnée d'une `\caption{}` en français bien détaillée telle qu'indiquée.
> - Tout le texte doit être en **français**.
> - Utiliser `inputenc` avec UTF-8 et `babel` avec `french`.
> - Générer une table des matières, une page de titre et des en-têtes de section claires.

---

## Page de Titre

- **Titre :** Documentation Technique DevOps — Architecture Microservices Spring Boot & Angular
- **Sous-titre :** Intégration Continue, Conteneurisation Docker et Supervision avec Prometheus
- **Date :** Février 2026

---

## Table des Matières

1. Introduction et Vue d'Ensemble du Projet
2. Architecture de l'Application
3. Conteneurisation avec Docker
4. Orchestration avec Docker Compose
5. Pipeline CI/CD avec Jenkins
6. Analyse de Qualité avec SonarQube
7. Supervision et Monitoring avec Prometheus
8. Configuration de l'API Gateway
9. Découverte de Services avec Eureka
10. Conclusion

---

## 1. Introduction et Vue d'Ensemble du Projet

Ce projet est une application web de gestion académique développée selon une architecture **microservices**. Il repose sur les technologies suivantes :

- **Backend :** Spring Boot (Java 11), Spring Cloud, Maven multi-modules
- **Frontend :** Angular 14, servi via Nginx
- **Conteneurisation :** Docker, Docker Compose
- **CI/CD :** Jenkins (pipeline déclaratif)
- **Qualité du code :** SonarQube avec JaCoCo
- **Supervision :** Prometheus (collecte des métriques via Spring Boot Actuator)

L'application permet de gérer des étudiants, des cours et des examens à travers une interface web Angular communiquant avec les microservices via une passerelle API (Spring Cloud Gateway).

---

## 2. Architecture de l'Application

### 2.1 Vue d'ensemble des services

L'application est composée des microservices suivants :

| Service            | Port interne | Port exposé | Base de données    | Rôle                                     |
|--------------------|-------------|-------------|---------------------|------------------------------------------|
| `eureka-service`   | 8761        | 18761       | —                   | Registre de découverte de services        |
| `user-service`     | 8080        | 18080       | PostgreSQL (5432)   | Gestion des étudiants/utilisateurs        |
| `course-service`   | 8081        | 18081       | MySQL (3306)        | Gestion des cours                         |
| `exam-service`     | 8082        | 18082       | MySQL (3306)        | Gestion des examens                       |
| `api-gateway`      | 8090        | 18090       | —                   | Routage et point d'entrée unifié          |
| `frontend`         | 80          | 4200        | —                   | Interface Angular servie par Nginx        |

### 2.2 Schéma d'architecture

<!-- IMAGE 1 -->
**[IMAGE : architecture-globale]**
*Légende : Schéma d'architecture globale de l'application — le frontend Angular communique avec l'API Gateway (port 18090), qui route les requêtes vers user-service, course-service et exam-service. Chaque service backend est enregistré auprès d'Eureka. PostgreSQL sert user-service et MySQL sert course-service et exam-service.*

### 2.3 Structure du dépôt

Le projet est organisé en deux grandes parties :

```
spring-boot-microservices-angular/
├── backend/                    # POM parent Maven multi-modules
│   ├── pom.xml                 # Déclaration des modules
│   ├── common-service/         # Bibliothèque partagée
│   ├── common-exam/            # DTOs communs examens
│   ├── common-student/         # DTOs communs étudiants
│   ├── eureka-service/         # Serveur de découverte Netflix Eureka
│   ├── user-service/           # Microservice utilisateurs (PostgreSQL)
│   ├── course-service/         # Microservice cours (MySQL)
│   ├── exam-service/           # Microservice examens (MySQL)
│   └── api-gateway-service/    # Passerelle API Spring Cloud Gateway
├── frontend/                   # Application Angular
│   ├── src/
│   ├── Dockerfile
│   └── nginx.conf
├── docker-compose.yml          # Orchestration de tous les services
├── prometheus.yml              # Configuration de scraping Prometheus
└── Jenkinsfile                 # Pipeline CI/CD déclaratif
```

### 2.4 Structure Maven multi-modules

Le fichier `backend/pom.xml` déclare un projet parent Maven regroupant tous les modules backend. Cela permet de partager les dépendances communes (Actuator, Micrometer/Prometheus, Lombok) et les configurations de build (JaCoCo, SonarQube) entre tous les services.

```xml
<modules>
    <module>common-service</module>
    <module>common-exam</module>
    <module>common-student</module>
    <module>course-service</module>
    <module>exam-service</module>
    <module>user-service</module>
    <module>eureka-service</module>
    <module>api-gateway-service</module>
</modules>
```

Les dépendances héritées par tous les modules :

```xml
<!-- Actuator : expose /actuator/health, /actuator/metrics, etc. -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!-- Micrometer Prometheus : expose /actuator/prometheus -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

---

## 3. Conteneurisation avec Docker

### 3.1 Stratégie de build multi-stage

Chaque microservice backend utilise un **Dockerfile multi-stage** pour produire des images légères et optimisées :

1. **Stage `builder` :** Utilise `maven:3.8.5-openjdk-11-slim` pour compiler et packager le JAR.
2. **Stage final :** Utilise `eclipse-temurin:11-jre` (JRE uniquement, sans JDK) pour exécuter l'application.

Cette approche réduit la taille finale de l'image en ne conservant que le JRE et le JAR compilé.

**Exemple — Dockerfile de `eureka-service` :**

```dockerfile
FROM maven:3.8.5-openjdk-11-slim AS builder
WORKDIR /build
COPY pom.xml ./pom.xml
RUN mvn -f /build/pom.xml install -N -DskipTests -q

COPY eureka-service ./eureka-service
RUN mvn -f /build/eureka-service/pom.xml package -DskipTests -q

FROM eclipse-temurin:11-jre
WORKDIR /app
COPY --from=builder /build/eureka-service/target/eureka-service-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8761
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Exemple — Dockerfile de `user-service`** (avec modules communs) :

```dockerfile
FROM maven:3.8.5-openjdk-11-slim AS builder
WORKDIR /build
COPY pom.xml ./pom.xml
RUN mvn -f /build/pom.xml install -N -DskipTests -q

COPY common-service ./common-service
COPY common-student ./common-student
RUN mvn -f /build/common-service/pom.xml install -DskipTests -q
RUN mvn -f /build/common-student/pom.xml install -DskipTests -q

COPY user-service ./user-service
RUN mvn -f /build/user-service/pom.xml package -DskipTests -q

FROM eclipse-temurin:11-jre
WORKDIR /app
COPY --from=builder /build/user-service/target/user-service-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

> Les services dépendant de modules communs (`common-service`, `common-exam`, `common-student`) installent ces modules dans le dépôt Maven local du conteneur de build avant de compiler le service principal.

### 3.2 Dockerfile du Frontend Angular

Le frontend utilise également un build multi-stage :

1. **Stage `builder` :** Utilise `node:18-alpine` pour installer les dépendances npm et générer le build de production Angular.
2. **Stage final :** Utilise `nginx:1.25-alpine` pour servir les fichiers statiques générés.

```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install --legacy-peer-deps && \
    npm install --save-dev @types/node@18.19.0 --legacy-peer-deps
COPY . .
ENV NODE_OPTIONS=--max-old-space-size=2048
RUN npm run build -- --configuration production

FROM nginx:1.25-alpine
RUN rm -rf /usr/share/nginx/html/*
COPY --from=builder /app/dist/demo /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 3.3 Configuration Nginx pour le Frontend

Le fichier `nginx.conf` configure Nginx pour :
- Servir l'application Angular en mode SPA (Single Page Application).
- Mettre en cache les ressources statiques (JS, CSS, images).

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # SPA Angular : rediriger toutes les routes vers index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache des ressources statiques
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

<!-- IMAGE 2 -->
**[IMAGE : docker-images-list]**
*Légende : Liste des images Docker construites pour le projet — affichage de la commande `docker images` montrant toutes les images des microservices (eureka-service, user-service, course-service, exam-service, api-gateway, frontend) avec leurs tailles respectives.*

---

## 4. Orchestration avec Docker Compose

### 4.1 Vue d'ensemble

Le fichier `docker-compose.yml` orchestre l'ensemble des services sur un réseau Docker bridge dédié nommé `microservices-net`. Il définit les volumes persistants pour les bases de données et configure les dépendances entre services via `depends_on`.

```yaml
networks:
  microservices-net:
    driver: bridge

volumes:
  postgres-data:
  mysql-data:
```

### 4.2 Services de bases de données

**PostgreSQL** — utilisé par `user-service` :

```yaml
postgres:
  image: postgres:15-alpine
  mem_limit: 256m
  environment:
    POSTGRES_DB: microservices_users
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: changeme
  ports:
    - "5432:5432"
  volumes:
    - postgres-data:/var/lib/postgresql/data
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres"]
    interval: 10s
    timeout: 5s
    retries: 5
```

**MySQL** — utilisé par `course-service` et `exam-service` :

```yaml
mysql:
  image: mysql:8.0
  mem_limit: 300m
  environment:
    MYSQL_ROOT_PASSWORD: password
    MYSQL_DATABASE: microservices_db
  ports:
    - "3306:3306"
  volumes:
    - mysql-data:/var/lib/mysql
    - ./microservices_db.sql:/docker-entrypoint-initdb.d/microservices_db.sql
  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "--password=password"]
    interval: 10s
    timeout: 5s
    retries: 10
```

> Le script SQL `microservices_db.sql` est monté dans le répertoire d'initialisation de MySQL, ce qui garantit la création automatique du schéma au premier démarrage.

### 4.3 Services applicatifs

Chaque service applicatif partage les caractéristiques suivantes dans Docker Compose :
- **`mem_limit: 256m`** : limitation mémoire pour contrôler la consommation des ressources.
- **`JAVA_TOOL_OPTIONS=-Xms32m -Xmx128m`** : limitation du heap JVM.
- **`restart: unless-stopped`** : redémarrage automatique en cas d'erreur.
- **`MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE=health,info,prometheus,metrics`** : exposition des endpoints Actuator pour le monitoring.

**Exemple — `user-service` :**

```yaml
user-service:
  build:
    context: ./backend
    dockerfile: user-service/Dockerfile
  mem_limit: 256m
  ports:
    - "18080:8080"
  environment:
    - SPRING_APPLICATION_NAME=user-service
    - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-service:8761/eureka
    - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/microservices_users
    - SPRING_DATASOURCE_USERNAME=postgres
    - SPRING_DATASOURCE_PASSWORD=changeme
    - JAVA_TOOL_OPTIONS=-Xms32m -Xmx128m
  depends_on:
    postgres:
      condition: service_started
    eureka-service:
      condition: service_started
```

### 4.4 Chaîne de dépendances des services

Le diagramme de dépendances `depends_on` entre services est le suivant :

```
postgres  ──────────────────────► user-service ──┐
mysql     ──► course-service ───────────────────► api-gateway ──► frontend
          └──► exam-service  ──────────────────────────────────►
eureka-service ──────────────────► (tous les services backend)
```

<!-- IMAGE 3 -->
**[IMAGE : docker-compose-ps]**
*Légende : Résultat de la commande `docker compose ps` montrant l'état de tous les conteneurs en cours d'exécution — statut `running`, ports mappés et état de santé (healthy/starting) pour chaque service.*

<!-- IMAGE 4 -->
**[IMAGE : docker-compose-logs]**
*Légende : Extrait des logs Docker Compose lors du démarrage des services — visible la séquence de démarrage : PostgreSQL, MySQL, Eureka, puis les microservices métier, et enfin l'API Gateway et le Frontend.*

---

## 5. Pipeline CI/CD avec Jenkins

### 5.1 Vue d'ensemble du pipeline

Le fichier `Jenkinsfile` définit un **pipeline déclaratif Jenkins** composé de 6 étapes successives. Il s'exécute sur un serveur Jenkins configuré avec accès à Docker et Maven.

```
Checkout ──► Build & Test ──► SonarQube Analysis ──► Quality Gate ──► Docker Build ──► Docker Up
```

**Variables d'environnement du pipeline :**

```groovy
environment {
    GITHUB_REPO_URL   = 'https://github.com/cheikhtourad19/spring-boot-microservices-angular.git'
    GITHUB_BRANCH     = 'main'
    SONAR_SERVER_NAME = 'SonarServer'
    SONAR_HOST_URL    = 'http://192.168.56.11:9000'
    SONAR_PROJECT_KEY = 'spring-boot-microservices'
    COMPOSE_FILE      = 'docker-compose.yml'
    COMPOSE_PROJECT   = 'microservices'
}
```

### 5.2 Étapes du Pipeline

#### Étape 1 — Checkout

Cette étape récupère le code source depuis GitHub. Elle gère intelligemment les deux cas : premier déploiement (clone) et mises à jour suivantes (pull).

```groovy
stage('Checkout') {
    steps {
        sh """
            if [ -d ".git" ]; then
                git fetch origin
                git reset --hard origin/main
                git clean -fd
            else
                git clone -b main ${env.GITHUB_REPO_URL} .
            fi
        """
    }
}
```

#### Étape 2 — Build & Test

Compilation et packaging de tous les modules Maven en mode batch :

```groovy
stage('Build & Test') {
    steps {
        dir('backend') {
            sh '''
                mvn clean verify \
                    --batch-mode \
                    --no-transfer-progress \
                    -DskipTests=true
            '''
        }
    }
}
```

#### Étape 3 — Analyse SonarQube

Soumission du code au serveur SonarQube pour analyse statique de la qualité :

```groovy
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv(env.SONAR_SERVER_NAME) {
            dir('backend') {
                sh """
                    mvn sonar:sonar \
                        --batch-mode \
                        --no-transfer-progress \
                        -Dsonar.projectKey=${env.SONAR_PROJECT_KEY} \
                        -Dsonar.projectName='${env.SONAR_PROJECT_NAME}'
                """
            }
        }
    }
}
```

#### Étape 4 — Quality Gate

Vérification des résultats de l'analyse SonarQube. L'URL du tableau de bord est affichée dans les logs Jenkins.

#### Étape 5 — Docker Compose Build

Construction de toutes les images Docker en parallèle avec l'option `--parallel` pour optimiser les temps de build :

```groovy
stage('Docker Compose Build') {
    steps {
        sh """
            docker compose \
                -p ${env.COMPOSE_PROJECT} \
                -f ${env.WORKSPACE}/${env.COMPOSE_FILE} \
                build --no-cache --parallel
        """
    }
}
```

#### Étape 6 — Docker Compose Up

Démarrage de l'ensemble des services en mode détaché avec suppression des conteneurs orphelins :

```groovy
stage('Docker Compose Up') {
    steps {
        sh """
            docker compose \
                -p ${env.COMPOSE_PROJECT} \
                -f ${env.WORKSPACE}/${env.COMPOSE_FILE} \
                up -d --remove-orphans
        """
    }
}
```

### 5.3 Actions Post-Build

```groovy
post {
    success  { echo "Pipeline RÉUSSI — Build #${env.BUILD_NUMBER}" }
    failure  { echo "Pipeline EN ÉCHEC — Build #${env.BUILD_NUMBER}" }
    unstable { echo "Pipeline INSTABLE — vérifier les rapports de test." }
    always   {
        // Nettoyage systématique : arrêt de tous les conteneurs
        sh "docker compose -p ${env.COMPOSE_PROJECT} down --volumes --remove-orphans || true"
    }
}
```

> **Note :** L'action `always` arrête et supprime tous les conteneurs après chaque exécution du pipeline (succès ou échec), ce qui garantit un environnement propre pour chaque build.

### 5.4 Options du pipeline

| Option | Valeur | Description |
|---|---|---|
| `buildDiscarder` | 10 derniers builds | Limitation de l'espace disque Jenkins |
| `timeout` | 60 minutes | Durée maximale d'exécution |
| `disableConcurrentBuilds` | activé | Pas d'exécutions parallèles |
| `skipDefaultCheckout` | activé | Checkout géré manuellement |

<!-- IMAGE 5 -->
**[IMAGE : jenkins-pipeline-view]**
*Légende : Vue du pipeline Jenkins dans l'interface Blue Ocean ou la vue classique — affichage des 6 étapes du pipeline (Checkout, Build & Test, SonarQube Analysis, Quality Gate, Docker Build, Docker Up) avec leur statut (succès/échec) et leur durée d'exécution.*

<!-- IMAGE 6 -->
**[IMAGE : jenkins-build-console]**
*Légende : Sortie console d'un build Jenkins réussi — visible les logs Maven (compilation des modules), puis la soumission à SonarQube, le build Docker parallèle et le démarrage des conteneurs avec `docker compose up`.*

<!-- IMAGE 7 -->
**[IMAGE : jenkins-job-history]**
*Légende : Historique des builds Jenkins pour le projet — liste des 10 derniers builds avec leur numéro, durée, statut (bleu = succès, rouge = échec) et horodatage.*

---

## 6. Analyse de Qualité avec SonarQube

### 6.1 Configuration dans le POM parent

Le plugin SonarQube est configuré dans le POM parent et activé via un profil Maven dédié :

```xml
<plugin>
    <groupId>org.sonarsource.scanner.maven</groupId>
    <artifactId>sonar-maven-plugin</artifactId>
    <version>3.11.0.3922</version>
</plugin>
```

**Propriétés SonarQube :**

```xml
<sonar.projectKey>spring-boot-microservices</sonar.projectKey>
<sonar.projectName>Spring Boot Microservices</sonar.projectName>
<sonar.host.url>http://localhost:9000</sonar.host.url>
<sonar.java.coveragePlugin>jacoco</sonar.java.coveragePlugin>
<sonar.coverage.jacoco.xmlReportPaths>
    ${project.build.directory}/site/jacoco/jacoco.xml
</sonar.coverage.jacoco.xmlReportPaths>
```

### 6.2 Couverture de code avec JaCoCo

JaCoCo est activé sur tous les modules pour générer des rapports de couverture de tests :

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <id>prepare-agent</id>
            <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>verify</phase>
            <goals><goal>report</goal></goals>
        </execution>
    </executions>
</plugin>
```

<!-- IMAGE 8 -->
**[IMAGE : sonarqube-dashboard]**
*Légende : Tableau de bord SonarQube du projet `spring-boot-microservices` — affichage des indicateurs clés : bugs, vulnérabilités, code smells, couverture de tests (JaCoCo), duplications de code et note de qualité globale (Quality Gate).*

<!-- IMAGE 9 -->
**[IMAGE : sonarqube-issues]**
*Légende : Vue détaillée des issues SonarQube — liste des problèmes de code détectés (bugs, vulnérabilités, code smells) avec leur sévérité, le fichier concerné et la ligne de code correspondante.*

---

## 7. Supervision et Monitoring avec Prometheus

### 7.1 Architecture de supervision

La supervision repose sur **Spring Boot Actuator** et **Micrometer** :

- Chaque microservice expose ses métriques au format Prometheus via l'endpoint `/actuator/prometheus`.
- **Prometheus** collecte ces métriques en interrogeant périodiquement (toutes les 15 secondes) les endpoints de chaque service.
- Prometheus s'exécute directement sur la VM hôte et accède aux services via les ports mappés sur l'IP `192.168.56.11`.

### 7.2 Configuration Prometheus

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'eureka-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['192.168.56.11:18761']

  - job_name: 'user-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['192.168.56.11:18080']

  - job_name: 'course-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['192.168.56.11:18081']

  - job_name: 'exam-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['192.168.56.11:18082']

  - job_name: 'api-gateway-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['192.168.56.11:18090']
```

### 7.3 Métriques exposées

Chaque service expose les métriques suivantes grâce à `spring-boot-starter-actuator` et `micrometer-registry-prometheus` :

| Endpoint | Description |
|---|---|
| `/actuator/health` | État de santé du service (UP/DOWN) |
| `/actuator/metrics` | Métriques JVM, HTTP, pool de connexions |
| `/actuator/prometheus` | Métriques au format Prometheus (scraping) |
| `/actuator/info` | Informations sur l'application |

Le tag `management.metrics.tags.application` permet d'identifier chaque service dans Prometheus :

```properties
management.metrics.tags.application=${spring.application.name}
```

### 7.4 Tableau des ports de monitoring

| Service          | Port hôte | Endpoint Prometheus                              |
|------------------|-----------|--------------------------------------------------|
| eureka-service   | 18761     | `http://192.168.56.11:18761/actuator/prometheus` |
| user-service     | 18080     | `http://192.168.56.11:18080/actuator/prometheus` |
| course-service   | 18081     | `http://192.168.56.11:18081/actuator/prometheus` |
| exam-service     | 18082     | `http://192.168.56.11:18082/actuator/prometheus` |
| api-gateway      | 18090     | `http://192.168.56.11:18090/actuator/prometheus` |

<!-- IMAGE 10 -->
**[IMAGE : prometheus-targets]**
*Légende : Interface web Prometheus (http://192.168.56.11:9090/targets) montrant l'état de tous les targets de scraping — chaque microservice apparaît avec son statut `UP` ou `DOWN`, l'heure du dernier scraping réussi et le label `application` correspondant.*

<!-- IMAGE 11 -->
**[IMAGE : prometheus-graph]**
*Légende : Interface de requête Prometheus — exemple de graphique de la métrique `http_server_requests_seconds_count` filtré par service (`application="user-service"`) montrant l'évolution du nombre de requêtes HTTP au fil du temps.*

<!-- IMAGE 12 -->
**[IMAGE : actuator-health-endpoint]**
*Légende : Réponse JSON de l'endpoint `/actuator/health` d'un microservice — affichage du statut global `UP`, avec les détails de l'état de santé de la base de données (diskSpace, ping).*

---

## 8. Configuration de l'API Gateway

### 8.1 Rôle de l'API Gateway

L'**API Gateway** (Spring Cloud Gateway) est le point d'entrée unique de l'application. Elle :
- Reçoit toutes les requêtes du frontend Angular.
- Route chaque requête vers le microservice approprié via le load balancer Eureka (`lb://`).
- Gère les en-têtes CORS pour autoriser les appels cross-origin depuis le frontend.
- Est enregistrée dans Eureka au même titre que les autres services.

### 8.2 Configuration des routes

```properties
# USER SERVICE
spring.cloud.gateway.routes[0].id=user-core
spring.cloud.gateway.routes[0].uri=lb://USER-SERVICE
spring.cloud.gateway.routes[0].predicates=Path=/students/**

# COURSE SERVICE
spring.cloud.gateway.routes[1].id=course-core
spring.cloud.gateway.routes[1].uri=lb://COURSE-SERVICE
spring.cloud.gateway.routes[1].predicates=Path=/courses/**

# EXAM SERVICE
spring.cloud.gateway.routes[2].id=exams-core
spring.cloud.gateway.routes[2].uri=lb://EXAM-SERVICE
spring.cloud.gateway.routes[2].predicates=Path=/exams/**

spring.cloud.loadbalancer.ribbon.enabled=false
```

### 8.3 Configuration CORS

```properties
spring.cloud.gateway.globalcors.cors-configurations.[/**].allowed-origin-patterns=*
spring.cloud.gateway.globalcors.cors-configurations.[/**].allowed-methods=GET,POST,PUT,DELETE,PATCH,OPTIONS,HEAD
spring.cloud.gateway.globalcors.cors-configurations.[/**].allowed-headers=*
spring.cloud.gateway.globalcors.cors-configurations.[/**].allow-credentials=true
spring.cloud.gateway.globalcors.cors-configurations.[/**].max-age=3600
spring.cloud.gateway.default-filters[0]=DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
```

### 8.4 Schéma de routage

```
Client Angular (port 4200)
        │
        ▼
  API Gateway (port 18090)
        │
        ├─── /students/**  ──► user-service   (lb://USER-SERVICE)
        ├─── /courses/**   ──► course-service (lb://COURSE-SERVICE)
        └─── /exams/**     ──► exam-service   (lb://EXAM-SERVICE)
```

<!-- IMAGE 13 -->
**[IMAGE : api-gateway-routes]**
*Légende : Test des routes de l'API Gateway via un outil comme Postman ou curl — exemple d'une requête GET vers `http://192.168.56.11:18090/students` retournant avec succès la liste des étudiants, démontrant le routage correct vers user-service.*

---

## 9. Découverte de Services avec Eureka

### 9.1 Configuration du serveur Eureka

Le service `eureka-service` est configuré en mode **standalone** (pas de réplication) :

```yaml
spring:
  application:
    name: eureka-service
server:
  port: 8761
eureka:
  client:
    registerWithEureka: false
    fetchRegistry: false
    service-url:
      default-zone: http://localhost:8761/eureka
```

### 9.2 Enregistrement des clients

Chaque microservice s'enregistre auprès d'Eureka via la variable d'environnement :

```
EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-service:8761/eureka
EUREKA_INSTANCE_PREFER_IP_ADDRESS=true
```

L'option `EUREKA_INSTANCE_PREFER_IP_ADDRESS=true` assure que les instances s'enregistrent avec leur adresse IP plutôt que leur nom d'hôte, ce qui est préférable dans un environnement Docker.

### 9.3 Healthcheck Eureka

```yaml
healthcheck:
  test: ["CMD-SHELL", "wget -qO- http://localhost:8761/actuator/health || exit 1"]
  interval: 15s
  timeout: 5s
  retries: 10
  start_period: 30s
```

<!-- IMAGE 14 -->
**[IMAGE : eureka-dashboard]**
*Légende : Tableau de bord Eureka (http://192.168.56.11:18761) montrant tous les services enregistrés — user-service, course-service, exam-service et api-gateway-service apparaissent avec le statut `UP` et leur adresse IP dans le réseau Docker.*

---

## 10. Conclusion

### 10.1 Récapitulatif de la chaîne DevOps

Ce projet met en œuvre une chaîne DevOps complète :

```
Développement
     │
     ▼
  GitHub (dépôt source)
     │
     ▼
  Jenkins (CI/CD)
     ├── Maven Build & Test
     ├── SonarQube (qualité du code + couverture JaCoCo)
     ├── Docker Build (images multi-stage)
     └── Docker Compose Deploy
              │
              ▼
     Environnement d'exécution
     ├── Services Spring Boot (Java 11)
     ├── Bases de données (PostgreSQL, MySQL)
     ├── Eureka (découverte de services)
     ├── API Gateway (Spring Cloud Gateway)
     ├── Frontend Angular (Nginx)
     └── Prometheus (supervision des métriques)
```

### 10.2 Points forts de l'architecture

| Aspect | Solution mise en œuvre |
|---|---|
| **Conteneurisation** | Docker multi-stage pour des images légères |
| **Orchestration** | Docker Compose avec réseau bridge dédié |
| **CI/CD** | Pipeline Jenkins déclaratif en 6 étapes |
| **Qualité** | SonarQube + JaCoCo pour l'analyse statique |
| **Découverte** | Netflix Eureka (Spring Cloud) |
| **Routage** | Spring Cloud Gateway avec load balancing |
| **Monitoring** | Prometheus + Spring Boot Actuator + Micrometer |
| **Performance** | Limitation mémoire JVM et Docker par service |

### 10.3 Résumé des ports exposés

| Service | Port exposé | URL d'accès |
|---|---|---|
| Frontend Angular | 4200 | `http://192.168.56.11:4200` |
| API Gateway | 18090 | `http://192.168.56.11:18090` |
| Eureka Dashboard | 18761 | `http://192.168.56.11:18761` |
| SonarQube | 9000 | `http://192.168.56.11:9000` |
| Prometheus | 9090 | `http://192.168.56.11:9090` |
| user-service | 18080 | `http://192.168.56.11:18080` |
| course-service | 18081 | `http://192.168.56.11:18081` |
| exam-service | 18082 | `http://192.168.56.11:18082` |

<!-- IMAGE 15 -->
**[IMAGE : application-frontend]**
*Légende : Interface principale de l'application Angular en cours d'exécution sur le port 4200 — affichage du menu de navigation avec les sections Étudiants, Cours et Examens, démontrant le bon fonctionnement de l'ensemble de la chaîne (Frontend → API Gateway → Microservices → Bases de données).*
