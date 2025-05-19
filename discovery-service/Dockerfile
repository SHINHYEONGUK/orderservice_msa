# 첫번째 스테이지 -> 빌드 영역
# 베이스 이미지로 openjdk17-slim 버전을 사용하겠다.
FROM openjdk:17-jdk-slim AS build

# 워킹 디렉토리를 /app으로 지정
WORKDIR /app

# 소스코드와 Gradle 관련 파일 모두 복사
COPY . .

# gradle wrapper로 빌드하자 -> 실행 권한 먼저 주기
RUN chmod +x ./gradlew

# 이전에 빌드한 내용 있으면 clean 하고 build해라 (테스트 제외)
RUN ./gradlew clean build -x test

# 두번째 스테이지 -> 실행 영역
FROM openjdk:17-jdk-slim

# 워킹 디렉토리를 /app으로 지정
WORKDIR /app

# 컨테이너에서 curl 사용을 위해 설치 후, 임시 패키지 캐시 제거
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# build라는 별칭으로 만들어진 스테이지에서
# *.jar 파일을 app.jar로 복사해서 이미지에 세팅
COPY --from=build /app/build/libs/*.jar app.jar

# (선택) 컨테이너 포트 8888 노출 (Spring Cloud Config 서버 등 포트)
EXPOSE 8888

# CMD는 기본 실행 명령어를 의미. 컨테이너 실행 시에 다른 명령어가 주어지면 그 명령어로 대체됨.
# ENTRYPOINT는 반드시 실행되어야 할 명령어를 의미. 다른 명령어로 대체되지 않음.
# 스프링 부트는 무조건 -jar 옵션으로 실행되어야 하기에 ENTRYPOINT로 안전하게 선언.
ENTRYPOINT ["java", "-jar", "app.jar"]
