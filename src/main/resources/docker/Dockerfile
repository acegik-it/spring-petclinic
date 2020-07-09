FROM openjdk:8-jre-alpine
RUN mkdir -p /usr/local/services
ADD spring-petclinic-*.jar /usr/local/services/spring-petclinic.jar
ADD run.sh docker-entrypoint.sh
RUN chmod +x docker-entrypoint.sh
ENTRYPOINT ["sh", "docker-entrypoint.sh"]
