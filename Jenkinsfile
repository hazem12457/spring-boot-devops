pipeline {
    agent any

    tools {
        jdk 'JAVA_HOME'
        maven 'M2_HOME'
    }

    environment {
        DOCKERHUB    = credentials('dockerhub-credentials')
        IMAGE_LOCAL  = 'backend-app:latest'
        IMAGE_REMOTE = 'hazem1247/backend-app:latest'
        NETWORK      = 'devops-net'
    }

    stages {
        stage('Git') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test Maven') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t $IMAGE_LOCAL .'
            }
        }

        stage('Docker Push') {
            steps {
                retry(3) {
                    sh '''
                        echo "$DOCKERHUB_PSW" | docker login -u "$DOCKERHUB_USR" --password-stdin
                        docker tag $IMAGE_LOCAL $IMAGE_REMOTE
                        docker push $IMAGE_REMOTE
                    '''
                }
            }
        }

        stage('Deploiement MySQL') {
            steps {
                sh '''
                    docker network create $NETWORK || true
                    if [ -z "$(docker ps -q -f name=^mysql$)" ]; then
                        docker rm -f mysql || true
                        docker run -d --name mysql --network $NETWORK \
                          -e MYSQL_ROOT_PASSWORD=root \
                          -e MYSQL_DATABASE=test_db \
                          -v mysql-cd-data:/var/lib/mysql \
                          mysql:8.0
                    fi
                    for i in $(seq 1 30); do
                        docker exec mysql mysqladmin ping -h localhost -proot --silent && break
                        sleep 3
                    done
                '''
            }
        }

        stage('Deploiement backend-app') {
            steps {
                sh 'docker rm -f backend-app || true'
                retry(3) {
                    sh 'docker pull $IMAGE_REMOTE'
                }
                sh '''
                    docker run -d --name backend-app --network $NETWORK \
                      -p 8090:8080 --restart on-failure \
                      -e SPRING_DATASOURCE_URL="jdbc:mysql://mysql:3306/test_db?createDatabaseIfNotExist=true" \
                      -e SPRING_DATASOURCE_USERNAME=root \
                      -e SPRING_DATASOURCE_PASSWORD=root \
                      $IMAGE_REMOTE
                '''
            }
        }

        stage('Verification') {
            steps {
                sh '''
                    for i in $(seq 1 20); do
                        docker logs backend-app 2>&1 | grep -q "Started BackendApplication" && break
                        sleep 3
                    done
                    docker ps
                    docker logs backend-app
                    docker logs backend-app 2>&1 | grep "Started BackendApplication"
                '''
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
    }
}
