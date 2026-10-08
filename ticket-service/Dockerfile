# ── Étape 1 : build ─────────────────────────────────────────────────────────
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app

# Copier le pom seul pour mettre les dépendances en cache
COPY pom.xml .
RUN mvn -q -B dependency:go-offline

# Copier les sources et compiler sans les tests
COPY src ./src
RUN mvn -q -B package -DskipTests

# ── Étape 2 : image finale (JRE seulement, non-root) ────────────────────────
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Utilisateur système non-root
RUN addgroup -S spring && adduser -S -u 10001 -G spring spring
USER spring

# Récupérer le jar produit à l'étape build
COPY --from=build /app/target/*.jar app.jar

EXPOSE 8086

ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "app.jar"]
