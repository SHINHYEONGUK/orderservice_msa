// Groovy에서만 인식되는 전역 변수 선언 (Jenkins environment와 별개임)
def ecrLoginHelper = "docker-credential-ecr-login"
def deployHost = "54.180.148.191" // 배포 대상 EC2 서버의 public/private IP

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
                        echo "Changed files: ${changedFiles}"

                        // 변경된 파일의 경로가 서비스 디렉토리명으로 시작하면 빌드 대상에 추가
                        serviceDirs.each { service ->
                            if (changedFiles.any { it.startsWith(service + "/") }) {
                                changedServices.add(service)
                            }
                        }
                    }

                    // 빌드/배포 대상 서비스명을 콤마로 합쳐 환경 변수로 저장(다음 stage에서 사용)
                    env.CHANGED_SERVICES = changedServices.join(",")
                    if (env.CHANGED_SERVICES == "") {
                        echo "No changes detected in service directories. Skipping build and deployment."
                        currentBuild.result = 'SUCCESS' // 변경 없음: 파이프라인 종료(성공)
                    }
                }
            }
        }

        // 3단계: 변경된 서비스만 빌드(gradlew clean build -x test)
        stage('Build Changed Services') {
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
                script {
                    // Jenkins credentials 플러그인으로 등록된 aws-key 자격증명 사용
                    withAWS(region: "${env.REGION}", credentials: "aws-key") {
                        def changedServices = env.CHANGED_SERVICES.split(",")
                        changedServices.each { service ->
                            sh """
                            # ECR 로그인 (실전에서는 credential helper 대신 AWS CLI 로그인이 더 일반적)
                            aws ecr get-login-password --region ${env.REGION} | docker login --username AWS --password-stdin ${env.ECR_URL}

                            # 도커 이미지 빌드 및 푸시
                            docker build -t ${service}:latest ${service}
                            docker tag ${service}:latest ${env.ECR_URL}/${service}:latest
                            docker push ${env.ECR_URL}/${service}:latest
                            """
                        }
                    }
                }
            }
        }

        // 5단계: EC2 서버로 docker-compose.yml 전송, 이미지 pull & 서비스 재시작
        stage('Deploy Changed Services to AWS EC2') {

            steps {
                // SSH Private Key(예: deploy-key)로 SSH/SCP 명령 실행
                sshagent(credentials: ["deploy-key"]) {
                    script {
                        // docker-compose는 여러 서비스를 띄울 때 공백으로 구분해야 함 (a b c ...)
                        def changedServiceList = env.CHANGED_SERVICES.replace(",", " ")
                        sh """
                        # 1. EC2 서버로 docker-compose.yml 파일 복사
                        scp -o StrictHostKeyChecking=no docker-compose.yml ubuntu@${deployHost}:/home/ubuntu/docker-compose.yml

                        # 2. EC2 서버에 접속해서 AWS 로그인 & 변경 서비스만 pull/up
                        ssh -o StrictHostKeyChecking=no ubuntu@${deployHost} "
                          aws ecr get-login-password --region ${env.REGION} | docker login --username AWS --password-stdin ${env.ECR_URL}
                          docker-compose pull ${changedServiceList}
                          docker-compose up -d ${changedServiceList}
                        "
                        """
                    }
                }
            }
        }

    }
}
