FROM openjdk:8-jdk-alpine
VOLUME /tmp
ADD sample-spring-boot-jms-2.0.0-rc3.jar sample-spring-boot-jms-2.0.0-rc3.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/sample-spring-boot-jms-2.0.0-rc3.jar"]
