pipeline {
    agent any

    // 1. Параметризованная сборка
    parameters {
        string(name: 'STUDENT_NAME', defaultValue: 'Иванов Иван', description: 'Ваше ФИО')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'], description: 'Среда')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Запускать тесты')
    }

    environment {
        // ВАЖНО: Замените на ваши данные!
        DOCKER_USER = 'stasyfreak'
        GITHUB_USER = '5taZ'
        
        DOCKER_IMAGE = "${DOCKER_USER}/student-app:${BUILD_NUMBER}"
        CONTAINER_NAME = "student-app-${params.ENVIRONMENT}"
    }

    stages {
        stage('Checkout') {
            steps {
                cleanWs()
                checkout scm
            }
        }

        // 2 и 3. Условные этапы и Unit + Integration тесты
        stage('Tests') {
            when { expression { params.RUN_TESTS == true } }
            stages {
                stage('Unit Tests') {
                    steps {
                        echo 'Запуск Unit тестов...'
                        sh '''
                            python3 -m venv venv
                            . venv/bin/activate
                            pip install -r requirements.txt
                            python3 -m unittest test_app.py -v
                        '''
                    }
                }
                stage('Integration Tests') {
                    steps {
                        echo 'Запуск интеграционных тестов (заглушка)...'
                    }
                }
            }
        }

        // 4. Сборка Docker-образа
        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${DOCKER_IMAGE}")
                }
            }
        }

        // 5. Push образа в Docker Hub
        stage('Push to Registry') {
            steps {
                script {
                    docker.withRegistry('', 'docker-hub-credentials') {
                        dockerImage.push()
                        dockerImage.push('latest')
                    }
                }
            }
        }

        // 6. Deploy в dev/staging без подтверждения
        stage('Deploy to Dev') {
            when { expression { params.ENVIRONMENT == 'dev' } }
            steps {
                script {
                    sh "docker rm -f ${CONTAINER_NAME} || true"
                    sh "docker run -d --name ${CONTAINER_NAME} -p 8081:5000 -e STUDENT_NAME='${params.STUDENT_NAME} (DEV)' ${DOCKER_IMAGE}"
                }
            }
        }

        stage('Deploy to Staging') {
            when { expression { params.ENVIRONMENT == 'staging' } }
            steps {
                script {
                    sh "docker rm -f ${CONTAINER_NAME} || true"
                    sh "docker run -d --name ${CONTAINER_NAME} -p 8082:5000 -e STUDENT_NAME='${params.STUDENT_NAME} (STAGING)' ${DOCKER_IMAGE}"
                }
            }
        }

        // 7. Подтверждение перед Production
        stage('Approve Production') {
            when { expression { params.ENVIRONMENT == 'production' } }
            steps {
                input message: "Подтвердите развертывание в PRODUCTION?", ok: "Развернуть"
            }
        }

        stage('Deploy to Production') {
            when { expression { params.ENVIRONMENT == 'production' } }
            steps {
                script {
                    sh "docker rm -f ${CONTAINER_NAME} || true"
                    sh "docker run -d --name ${CONTAINER_NAME} -p 8083:5000 -e STUDENT_NAME='${params.STUDENT_NAME} (PROD)' ${DOCKER_IMAGE}"
                }
            }
        }

        // 8. Создание Git-тега при релизе
        stage('Tag Release') {
            when { expression { params.ENVIRONMENT == 'production' } }
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-credentials', passwordVariable: 'GIT_PASS', usernameVariable: 'GIT_USER')]) {
                    sh """
                        git config user.email "jenkins@example.com"
                        git config user.name "Jenkins"
                        git tag -a v1.0.${BUILD_NUMBER} -m "Release v1.0.${BUILD_NUMBER}"
                        git push https://${GIT_USER}:${GIT_PASS}@github.com/${GITHUB_USER}/simple-python-app.git v1.0.${BUILD_NUMBER}
                    """
                }
            }
        }
    }

    // 9 и 10. Уведомление и очистка рабочего пространства
    post {
        always {
            cleanWs()
        }
        success {
            echo "✅ УСПЕШНО: Пайплайн завершен для среды ${params.ENVIRONMENT}!"
        }
        failure {
            echo "❌ ОШИБКА: Пайплайн упал. Проверьте логи."
        }
    }
}
