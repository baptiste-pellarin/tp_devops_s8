FROM openjdk:11
WORKDIR /build
COPY Main.java .

RUN javac Main.java

FROM openjdk:11-jre
WORKDIR /app
COPY --from=0 /build ./


CMD ["java", "Main"]