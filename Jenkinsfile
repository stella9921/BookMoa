pipeline {
    agent any

    environment {
        // ✅ 네 실제 GCP / Jenkins 설정에 맞게 수정한 값들
        PROJECT_ID     = 'firm-aria-479509-m0'        // GCP 프로젝트 ID
        CLUSTER_NAME   = 'kube'                       // GKE 클러스터 이름 (gke-kube-... 노드들)
        LOCATION       = 'asia-northeast3-a'          // 클러스터가 떠 있는 zone
        CREDENTIALS_ID = 'gke'                        // Jenkins 크리덴셜 ID (ID 칸에 적은 값)
        DOCKER_IMAGE   = 'stella9921/bookmoa-backend' // Docker Hub 리포지토리 이름
    }

    stages {

        stage('Checkout code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker image') {
            steps {
                script {
                    echo 'Building Docker image...'
                    app = docker.build("${DOCKER_IMAGE}:${env.BUILD_ID}")
                }
            }
        }

        stage('Push Docker image') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerhub') {
                        echo 'Pushing Docker image...'
                        app.push('latest')
                        app.push("${env.BUILD_ID}")
                    }
                }
            }
        }

        stage('Deploy to GKE') {
            when {
                branch 'main'
            }
            steps {
                echo 'Deploying to GKE...'

                // 1) deployment.yaml 안 image 태그를 이번 빌드 이미지로 교체
                sh """
                    sed -i 's|image:.*|image: ${DOCKER_IMAGE}:${env.BUILD_ID}|' k8s/deployment.yaml
                    echo '--- DEBUG: image line in deployment.yaml ---'
                    grep 'image:' k8s/deployment.yaml
                """

                // 2) deployment.yaml 적용
                step([
                    $class: 'KubernetesEngineBuilder',
                    projectId: PROJECT_ID,
                    clusterName: CLUSTER_NAME,
                    location: LOCATION,
                    manifestPattern: 'k8s/deployment.yaml',
                    credentialsId: CREDENTIALS_ID,
                    verifyDeployments: true
                ])

                // 3) service.yaml 적용
                step([
                    $class: 'KubernetesEngineBuilder',
                    projectId: PROJECT_ID,
                    clusterName: CLUSTER_NAME,
                    location: LOCATION,
                    manifestPattern: 'k8s/service.yaml',
                    credentialsId: CREDENTIALS_ID,
                    verifyDeployments: true
                ])
            }
        }
    }
}
