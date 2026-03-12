# openjdk8 base image
FROM eclipse-temurin:8-jre

# directory 생성
RUN mkdir -p /usr/app/msa-attach-volume/messages
# jar 파일이 복사되는 위치
# 변수를 지정
ENV APP_HOME=/usr/app/
# 작업 시작 위치
WORKDIR $APP_HOME
# jar 파일 복사
COPY build/libs/*.jar apigateway.jar
# application port
EXPOSE 8000
# 실행
# -Dspring.profiles.active=${profile:default} -> .yaml에 있는 설정 값을 의미
CMD ["java", "-Dspring.profiles.active=${profile:default}", "-jar", "apigateway.jar"]
