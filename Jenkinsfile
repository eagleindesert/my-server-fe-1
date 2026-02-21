pipeline {
    agent any

    environment {
        // 도커 허브 또는 레지스트리 경로
        IMAGE_NAME = 'eagleindesert/fe-my-server-1'
        // 방금 Flux가 만들어준 매니페스트 전용 저장소 주소 (본인의 실제 주소로 변경 필요)
        MANIFEST_REPO_URL = 'https://github.com/eagleindesert/my-server-manifests.git'
    }

    stages {
        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} ."
                    sh "docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    // 회원님께서 입력해주신 Docker Hub credentialsId 적용
                    withDockerRegistry(credentialsId: 'token-for-Dockerhub-CICD-pipeline', url: '') {
                        sh "docker push ${IMAGE_NAME}:${BUILD_NUMBER}"
                        sh "docker push ${IMAGE_NAME}:latest"
                    }
                }
            }
        }

        stage('Update Manifest in GitOps Repo') {
            steps {
                // Jenkins 관리자에서 GitHub 접근 토큰을 'git-credentials'라는 이름으로 미리 생성해야 합니다
                withCredentials([gitUsernamePassword(credentialsId: 'token-for-Github-CICD-pipeline', gitT-from-26oolName: 'Default')]) {
                    sh """
                        # 1. 매니페스트 저장소 파기 후 새로 가져오기 (중복 에러 방지)
                        rm -rf manifests
                        git clone ${MANIFEST_REPO_URL} manifests
                        cd manifests
                        
                        # Git 사용자 셋팅 (Jenkins 역할)
                        git config user.name "eagleindesert"
                        git config user.email "toridoremi@naver.com"
                        
                        # 2. Linux의 sed 명령어로 yaml 파일 안의 이미지 태그 부분 찾아서 바꾸기
                        # 중요: clusters/my-cluster/deployment.yaml 경로가 실제 매니/페스트 경로와 일치해야 합니다.
                        sed -i "s|image: ${IMAGE_NAME}:.*|image: ${IMAGE_NAME}:${BUILD_NUMBER}|g" clusters/my-cluster/deployment.yaml
                        
                        # 3. 바뀐 파일 Git에 커밋하고 푸시하기
                        git add clusters/my-cluster/deployment.yaml
                        git commit -m "Update frontend image to build #${BUILD_NUMBER}"
                        git push origin main
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully. Flux will pull the new changes soon!'
        }
        failure {
            echo 'Build or Manifest update failed.'
        }
    }
}
