FROM anapsix/alpine-java:8_server-jre

ADD maven/*.jar app.jar

EXPOSE 8080

CMD java -server -jar app.jar
