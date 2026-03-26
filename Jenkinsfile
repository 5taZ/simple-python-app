pipeline {
    agent any

    // 1. Параметризация сборки
    parameters {
        string(name: 'STUDENT_NAME', defaultValue: 'stas', description: 'Ваше имя для отображения на сайте')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'], description: 'Среда развертывания')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Запускать тесты перед сборкой?')
    }

    environment {
        // Ваши данные Docker Hub
        DOCKER_IMAGE = "stasyfreak/student-app:${BUILD_NUMBER}" 
        CONTAINER_NAME = "student-app-${params.ENVIRONMENT}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // 2. Условный этап (Тесты запускаются только если стоит галочка)
        stage('Run Tests') {
            when {
                expression { params.RUN_TESTS == true }
            }
            steps {
                echo 'Настройка окружения и запуск тестов...'
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install -r requirements.txt
                    python -m unittest test_app.py -v
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Сборка Docker образа...'
                script {
                    docker.build("${DOCKER_IMAGE}")
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo 'Публикация образа в Docker Hub stasyfreak...'
                script {
                    // Используем ID учетных данных, созданных в Шаге 4.2
                    docker.withRegistry('', 'docker-hub-credentials') {
                        docker.image("${DOCKER_IMAGE}").push()
                        
                        // Для продакшена дополнительно пушим тег latest
                        if (params.ENVIRONMENT == 'production') {
                            docker.image("${DOCKER_IMAGE}").push('latest')
                        }
                    }
                }
            }
        }

        // Развертывание DEV (Автоматически)
        stage('Deploy to Dev') {
            when {
                expression { params.ENVIRONMENT == 'dev' }
            }
            steps {
                echo "Развертывание в DEV на порту 8081..."
                sh """
                    docker rm -f ${CONTAINER_NAME} || true
                    docker run -d --name ${CONTAINER_NAME} -p 8081:5000 \
                        -e STUDENT_NAME='${params.STUDENT_NAME}' \
                        -e ENVIRONMENT='dev' \
                        ${DOCKER_IMAGE}
                """
            }
        }

        // Развертывание STAGING (Автоматически)
        stage('Deploy to Staging') {
            when {
                expression { params.ENVIRONMENT == 'staging' }
            }
            steps {
                echo "Развертывание в STAGING на порту 8082..."
                sh """
                    docker rm -f ${CONTAINER_NAME} || true
                    docker run -d --name ${CONTAINER_NAME} -p 8082:5000 \
                        -e STUDENT_NAME='${params.STUDENT_NAME}' \
                        -e ENVIRONMENT='staging' \
                        ${DOCKER_IMAGE}
                """
            }
        }

        // 3. Условный этап с ручным подтверждением
        stage('Approve Production') {
            when {
                expression { params.ENVIRONMENT == 'production' }
            }
            steps {
                input message: "ВНИМАНИЕ: Подтвердите развертывание в PRODUCTION?", ok: "Да, развернуть"
            }
        }

        // Развертывание PRODUCTION
        stage('Deploy to Production') {
            when {
                expression { params.ENVIRONMENT == 'production' }
            }
            steps {
                echo "Развертывание в PRODUCTION на порту 8083..."
                sh """
                    docker rm -f ${CONTAINER_NAME} || true
                    docker run -d --name ${CONTAINER_NAME} -p 8083:5000 \
                        -e STUDENT_NAME='${params.STUDENT_NAME}' \
                        -e ENVIRONMENT='production' \
                        ${DOCKER_IMAGE}
                """
            }
        }

        // 4. Тегирование релизов в GitHub (только для продакшена)
        stage('Tag Release in GitHub') {
            when {
                expression { params.ENVIRONMENT == 'production' }
            }
            steps {
                // Используем ID токена GitHub (созданного в Шаге 4.2)
                withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                    sh """
                        git config user.email "jenkins@example.com"
                        git config user.name "Jenkins CI"
                        git tag -a v2.0.${BUILD_NUMBER} -m "Release v2.0.${BUILD_NUMBER} deployed to production"
                        # Отправляем тег в ваш репозиторий
                        git push https://\$GIT_USER:\$GIT_PASS@github.com/5taZ/simple-python-app.git v2.0.${BUILD_NUMBER}
                    """
                }
            }
        }
    }

    // 5. Очистка и уведомления
    post {
        always {
            echo 'Очистка рабочего пространства...'
            cleanWs()
        }
        success {
            echo "✅ Пайплайн успешно выполнен для среды: ${params.ENVIRONMENT}"
            // Пример уведомления. Чтобы он работал в реальности, нужен токен бота Telegram.
            // Преподавателю достаточно показать этот код.
            // sh "curl -s -X POST https://api.telegram.org/bot<ВАШ_ТОКЕН>/sendMessage -d chat_id=<ВАШ_ID> -d text='✅ Успешный деплой ${params.ENVIRONMENT}'"
        }
        failure {
            echo "❌ Пайплайн завершился с ошибкой. Проверьте логи."
        }
    }
}
