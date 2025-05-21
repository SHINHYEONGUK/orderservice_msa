// Groovy에서만 인식되는 전역 변수 선언 (Jenkins environment와 별개임)
def ecrLoginHelper = "docker-credential-ecr-login"
def deployHost = "54.180.148.191" // 배포 대상 EC2 서버의 public/private IP

// 젠킨스의 선언형 파이프라인 정의부 시작 (그루비 언어)
pipeline {
    agent any // 어떤 Jenkins 에이전트에서도 실행 가능

    // Jenkins가 관리하는 환경 변수 선언 (모든 stage에서 접근 가능)
    environment {
        SERVICE_DIRS = "config-service,discovery-service,gateway-service,user-service,ordering-service,product-service" // 서비스 폴더 목록(쉼표로 구분)
        ECR_URL = "221082190600.dkr.ecr.ap-northeast-2.amazonaws.com" // ECR(Elastic Container Registry) 엔드포인트
        REGION = "ap-northeast-2" // AWS 리전
    }

    stages {

        // 1단계: GitHub에서 소스코드를 체크아웃(다운로드)
        stage('Pull Codes from Github') {
            steps {
                checkout scm // Jenkins와 연결된 SCM(소스 제어: git 등)에서 코드 받기
            }
        }

        stage('Add Secret To Config-service') {
            steps {
                withCredentials([file(credentialsId: 'config-secret', variable:'configSecret' )]) {
                    script {
                        sh 'cp $configSecret config-service/src/main/resources/application-dev.yml'
                    }
                }
            }
        }

        // 2단계: 어떤 서비스가 변경됐는지 감지 (최초 커밋이면 전체 빌드)
        stage('Detect Changes') {
            steps {
                script {
                    // 전체 커밋 수 확인 (최초 빌드인지 체크)
                    def commitCount = sh(script: "git rev-list --count HEAD", returnStdout: true).trim().toInteger()
                    def changedServices = []
                    def serviceDirs = env.SERVICE_DIRS.split(",")

                    if (commitCount == 1) {
                        // 첫 커밋이면 모든 서비스 빌드 대상으로 추가
                        echo "Initial commit detected. All services will be built."
                        changedServices = serviceDirs
                    } else {
                        // 변경된 파일 목록 가져오기 (마지막 커밋 1개 기준)
                        def changedFiles = sh(script: "git diff --name-only HEAD~1 HEAD", returnStdout: true).trim().split('\n')

                        // 변경된 파일 출력
                        // [user-service/src/main/resources/application.yml,
                        // user-service/src/main/java/com/playdata/userservice/controller/UserController.java,
                        // ordering-service/src/main/resources/application.yml]
                        echo "Changed files: ${changedFiles}"

                        // 변경된 파일의 경로가 서비스 디렉토리명으로 시작하면 빌드 대상에 추가
                        serviceDirs.each { service ->
                            // changedFiles라는 리스트를 조회해서 service 변수에 들어온 서비스 이름과
                            // 하나라도 일치하는 이름이 있다면 true, 하나도 존재하지 않으면 false
                            if (changedFiles.any { it.startsWith(service + "/") }) {
                                changedServices.add(service)
                            }
                        }
                    }

                    //변경된 서비스 이름을 모아놓은 리스트를 다른 스테이지에서도 사용하기 위해 환경 변수로 선언.
                    // join() -> 지정한 문자열을 구분자로 하여 리스트 요소를 하나의 문자열로 리턴. 중복 제거.
                    // 환경변수는 문자열만 선언할 수 있어서 join을 사용함.
                    // 빌드/배포 대상 서비스명을 콤마로 합쳐 환경 변수로 저장(다음 stage에서 사용)
                    env.CHANGED_SERVICES = changedServices.join(",")
                    if (env.CHANGED_SERVICES == "") {
                        echo "No changes detected in service directories. Skipping build and deployment."
                         // 성공 상태로 파이프라인을 종료
                        currentBuild.result = 'SUCCESS' // 변경 없음: 파이프라인 종료(성공)
                    }
                }
            }
        }

        // 3단계: 변경된 서비스만 빌드(gradlew clean build -x test)
        stage('Build Changed Services') {
            // 이 스테이지는 빌드되어야 할 서비스가 존재한다면 실행되는 스테이지.
            // 이전 스테이지에서 세팅한 CHANGED_SERVICES라는 환경변수가 비어있지 않아야만 실행.
            when {
                expression { env.CHANGED_SERVICES != "" } // 빌드 대상이 있을 때만 실행
            }
            steps {
                script {
                    def changedServices = env.CHANGED_SERVICES.split(",")
                    changedServices.each { service ->
                        sh """
                        echo "Building ${service}..."
                        cd ${service}
                        ./gradlew clean build -x test
                        ls -al ./build/libs
                        cd ..
                        """
                    }
                }
            }
        }

        // 4단계: Docker 이미지 빌드 및 AWS ECR에 Push
        stage('Build Docker Image & Push to AWS ECR') {
            when {
                expression { env.CHANGED_SERVICES != "" }
            }
            steps {
                     // jenkins에 저장된 credentials를 사용하여 AWS 자격증명을 설정.
                    withAWS(region: "${REGION}", credentials: "aws-key") {
                        def changedServices = env.CHANGED_SERVICES.split(",")
                        changedServices.each { service ->
                            sh """
                            # ECR에 이미지를 push하기 위해 인증 정보를 대신 검증해 주는 도구 다운로드.
                            # /usr/local/bin/ 경로에 해당 파일을 이동
                            curl -O https://amazon-ecr-credential-helper-releases.s3.us-east-2.amazonaws.com/0.4.0/linux-amd64/${ecrLoginHelper}
                            chmod +x ${ecrLoginHelper}
                            mv ${ecrLoginHelper} /usr/local/bin/

                            # Docker에게 push 명령을 내리면 지정된 URL로 push할 수 있게 설정.
                            # 자동으로 로그인 도구를 쓰게 설정
                            mkdir -p ~/.docker
                            echo '{"credHelpers": {"${ECR_URL}": "ecr-login"}}' > ~/.docker/config.json

                            docker build -t ${service}:latest ${service}
                            docker tag ${service}:latest ${ECR_URL}/${service}:latest
                            docker push ${ECR_URL}/${service}:latest
                            """
                        }
                    }

                }
            }
        }

        // 5단계: EC2 서버로 docker-compose.yml 전송, 이미지 pull & 서비스 재시작
        stage('Deploy Changed Services to AWS EC2') {

            steps {
                sshagent(credentials: ["deploy-key"]) {
                    sh """
                    # Jenkins에서 배포 서버로 docker-compose.yml 복사 후 전송
                    scp -o StrictHostKeyChecking=no docker-compose.yml ubuntu@${deployHost}:/home/ubuntu/docker-compose.yml

                    # 배포 서버로 직접 접속 시도 (compose 돌리러 갑니다!)
                    ssh -o StrictHostKeyChecking=no ubuntu@${deployHost} '
                    cd /home/ubuntu && \

                    # 시간이 지나 로그인 만료 시 필요한 명령
                    aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ECR_URL} && \

                    # docker compose를 이용해서 변경된 서비스만 이미지를 pull -> 일괄 실행
                    docker-compose pull ${env.CHANGED_SERVICES} && \
                    docker compose up -d ${env.CHANGED_SERVICES}
                    '
                    """
                }
            }
        }

    }
}
