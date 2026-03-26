pipeline {
    agent any

    // 1. Параметризация (Требование "Отлично")
    parameters {
        string(name: 'STUDENT_NAME', defaultValue: 'Иванов Иван', description: 'ФИО студента')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'], description: 'Среда развертывания')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Запускать тесты')
    }

    environment {
        // Укажите ВАШ логин от Docker Hub вместо yourdockerhub
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_IMAGE = "stasyfreak/student-app:${BUILD_NUMBER}" 
        CONTAINER_NAME = "student-app-${params.ENVIRONMENT}"
        // Порты для разных сред во избежание конфликтов
        DEV_PORT = '5000'
        STAGING_PORT = '5001'
        PROD_PORT = '8080'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Клонирование репозитория...'
                checkout scm
            }
        }

        stage('Tests') {
            when {
                expression { params.RUN_TESTS == true }
            }
            steps {
                echo 'Настройка окружения Python и запуск тестов...'
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    python -m unittest test_app.py -v
                '''
            }
            post {
                success {
                    // Создаем фиктивный xml для JUnit плагина, чтобы он работал (опционально)
                    echo 'Тесты успешно пройдены!'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Сборка Docker образа...'
                script {
                    dockerImage = docker.build("${DOCKER_IMAGE}")
                }
            }
        }

        stage('Push to Registry') {
            steps {
                echo 'Публикация образа в Docker Hub...'
                script {
                    // Используем ID credentials: docker-hub-credentials (убедитесь, что создали его в Jenkins)
                    docker.withRegistry('', 'docker-hub-credentials') {
                        dockerImage.push()
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            when { expression { params.ENVIRONMENT == 'dev' } }
            steps {
                echo "Развертывание в DEV..."
                script {
                    sh "docker rm -f ${CONTAINER_NAME} || true"
                    sh """
                        docker run -d --name ${CONTAINER_NAME} \
                        -p ${DEV_PORT}:5000 \
                        -e STUDENT_NAME='${params.STUDENT_NAME} (DEV)' \
                        ${DOCKER_IMAGE}
                    """
                }
            }
        }

        stage('Deploy to Staging') {
            when { expression { params.ENVIRONMENT == 'staging' } }
            steps {
                echo "Развертывание в STAGING..."
                script {
                    sh "docker rm -f ${CONTAINER_NAME} || true"
                    sh """
                        docker run -d --name ${CONTAINER_NAME} \
                        -p ${STAGING_PORT}:5000 \
                        -e STUDENT_NAME='${params.STUDENT_NAME} (STAGING)' \
                        ${DOCKER_IMAGE}
                    """
                }
            }
        }

        // 2. Условный этап с подтверждением (Требование "Отлично")
        stage('Approve Production') {
            when { expression { params.ENVIRONMENT == 'production' } }
            steps {
                script {
                    def userInput = input(
                        id: 'DeployProd', 
                        message: 'Подтвердите развертывание в PRODUCTION?', 
                        ok: 'Да, развернуть',
                        parameters: [
                            string(name: 'PRODUCTION_VERSION', defaultValue: "v1.0.${BUILD_NUMBER}", description: 'Версия релиза (Git Tag)')
                        ]
                    )
                    env.RELEASE_TAG = userInput
                }
                echo "Развертывание в PRODUCTION одобрено!"
            }
        }

        stage('Deploy to Production') {
            when { expression { params.ENVIRONMENT == 'production' } }
            steps {
                echo "Развертывание в PROD..."
                script {
                    sh "docker rm -f ${CONTAINER_NAME} || true"
                    sh """
                        docker run -d --name ${CONTAINER_NAME} \
                        -p ${PROD_PORT}:5000 \
                        -e STUDENT_NAME='${params.STUDENT_NAME} (PROD)' \
                        ${DOCKER_IMAGE}
                    """
                }
            }
        }

        // 3. Тегирование версий в Git (Требование "Отлично")
        stage('Tag Release in GitHub') {
            when { expression { params.ENVIRONMENT == 'production' } }
            steps {
                echo "Создание Git тега ${env.RELEASE_TAG}..."
                withCredentials([usernamePassword(credentialsId: 'github-credentials', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                    sh """
                        git config user.email "jenkins@example.com"
                        git config user.name "Jenkins CI"
                        git tag -a ${env.RELEASE_TAG} -m "Release ${env.RELEASE_TAG} deployed to Production"
                        git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/5taZ/simple-python-app.git ${env.RELEASE_TAG}
                    """
                }
            }
        }
    }

    // 4. Отправка уведомлений и очистка (Требование "Отлично")
    post {
        always {
            echo 'Очистка рабочего пространства...'
            cleanWs()
        }
        success {
            echo "Пайплайн успешно выполнен для ${params.ENVIRONMENT}!"
            // Пример уведомления по Email (требует настройки SMTP в Jenkins)
            /*
            emailext (
                to: 'your-email@example.com',
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "Сборка успешна. Среда: ${params.ENVIRONMENT}"
            )
            */
            
            // ПРОСТОЙ ВАРИАНТ ДЛЯ СДАЧИ (Telegram). 
            // Создайте бота в @BotFather, узнайте свой CHAT_ID и раскомментируйте:
            /*
            sh """
                curl -s -X POST https://api.telegram.org/bot<ВАШ_ТОКЕН>/sendMessage \
                -d chat_id=<ВАШ_CHAT_ID> \
                -d text="✅ Сборка ${env.JOB_NAME} #${env.BUILD_NUMBER} успешно развернута в ${params.ENVIRONMENT}!"
            """
            */
        }
        failure {
            echo "❌ Пайплайн завершился с ошибкой. Проверьте логи."
        }
    }
}
