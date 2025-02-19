pipeline {
    agent any
    environment {
        PROJECT_NAME = "kube-news"
        REPOSITORY = "gabriel2012rissi/${env.PROJECT_NAME}"
        VERSION = "v1.0.0"
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    // Obtendo o hash do commit atual
                    COMMIT = "${GIT_COMMIT.substring(0,8)}"
                    if ("${env.BRANCH_NAME}" == "master") {
                        IMAGE_TAG = "${env.VERSION}"
                    } else {
                        IMAGE_TAG = "${env.BRANCH_NAME}"
                    }
                }
            }
        }
        stage('Static Analysis') {
            environment {
                sonarHome = tool 'SONAR_SCANNER'
            }
            steps {
                // SonarQube Scanner
                sh """
                   ${sonarHome}/bin/sonar-scanner -e \
                   -Dsonar.projectKey=${env.PROJECT_NAME} \
                   -Dsonar.branch.name=${env.BRANCH_NAME} \
                   -Dsonar.sources=. \
                   -Dsonar.host.url=http://sonarqube:9000 \
                   -Dsonar.login=275ebb556ebdcf5f6ce6ad67e2272a8e6ada4faf
                   """
            }
        }
        stage('Build Application') {
            steps {
                script {
                    appImage = docker.build("${env.REPOSITORY}:${COMMIT}", "./src")
                }
            }
        }
        stage('Run Application') {
            steps {
                script {
                    // Criando a rede 'jenkins_test'
                    sh "docker network create jenkins_test-${env.BUILD_NUMBER} || true"

                    // Criando um banco de dados 'postgres'
                    sh """
                        docker run \
                        --detach \
                        --name postgres-${env.BUILD_NUMBER} \
                        --publish 5432:5432 \
                        -e POSTGRES_USER=test \
                        -e POSTGRES_PASSWORD=test123 \
                        -e POSTGRES_DB=db_test \
                        --network jenkins_test-${env.BUILD_NUMBER} \
                        --network-alias postgres \
                        postgres:alpine
                        """

                    // Aguardando até o banco de dados ser iniciado
                    sleep 30

                    // Criando o container 'kube-news'
                    sh """
                       docker run \
                       --detach \
                       --name kube-news-${env.BUILD_NUMBER} \
                       --publish 3000:3000 \
                       -e DB_HOST=postgres \
                       -e DB_USERNAME=test \
                       -e DB_PASSWORD=test123 \
                       -e DB_DATABASE=db_test \
                       --network jenkins_test-${env.BUILD_NUMBER} \
                       --network-alias kubenews \
                       ${appImage.id}
                       """

                    // Aguardando até o container ser iniciado
                    sleep 35

                    // Testando a conexao via curl
                    sh """
                       docker run \
                       --rm \
                       --network jenkins_test-${env.BUILD_NUMBER} \
                       --network-alias curl \
                       alpine/curl \
                       /bin/sh -c 'curl -iL -X GET http://kubenews:3000/ready'
                       """
                }
            }
            post {
                always {
                    echo 'Cleanning up...'

                    // Removendo o banco de dados 'postgres'
                    sh "docker stop postgres-${env.BUILD_NUMBER} || true"
                    sh "docker rm -f postgres-${env.BUILD_NUMBER} || true"

                    // Removendo o container 'kube-news'
                    sh "docker stop kube-news-${env.BUILD_NUMBER} || true"
                    sh "docker rm -f kube-news-${env.BUILD_NUMBER} || true"

                    sleep 10

                    // Removendo a rede 'jenkins_test'
                    sh "docker network rm jenkins_test-${env.BUILD_NUMBER} || true"
                }
            }
        }
        stage('Push to Docker Hub') {
            steps {
                script {
                    // Enviar as imagens para o repositório do Docker Hub
                    docker.withRegistry("https://registry.hub.docker.com", "dockerhub") {
                        if ("${env.BRANCH_NAME}" == "master") {
                            appImage.push("latest")
                            appImage.push("${IMAGE_TAG}")
                        } else {
                            appImage.push("${IMAGE_TAG}")
                        }
                    }
                }
            }
        }
        stage('Trigger Manifest Update') {
            steps {
                script {
                    echo 'Triggering Manifest Update...'
                    build job: 'kube-news-manifest-update', parameters: [
                        string(name: "DOCKER_IMAGE", value: "${env.REPOSITORY}:${IMAGE_TAG}")
                    ]
                }
            }
        }
    }
}