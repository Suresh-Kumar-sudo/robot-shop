#---------- Build stage --------------

FROM maven:3.9-eclipse-temurin-8 AS build

WORKDIR /opt/shipping

COPY pom.xml .
RUN  mvn dependency:go-offline

COPY src ./src
RUN mvn clean package -DskipTests

#------------ Run stage ------------

FROM eclipse-temurin:8-jre

WORKDIR /opt/shipping

RUN useradd -m appuser

ENV CART_ENDPOINT=cart:8080
ENV DB_HOST=mysql

COPY --from=build /opt/shipping/target/shipping-1.0.jar shipping.jar

USER appuser

EXPOSE 8080

HEALTHCHECK CMD curl -f http://localhost:8080/health || exit 1

CMD ["java", "-jar", "shipping.jar"]
