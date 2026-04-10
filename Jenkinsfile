pipeline {
    agent any

    parameters {
        string(name: 'STUDENT_NAME', defaultValue: 'Полисимоненко', description: 'ФИО студента')
        string(name: 'PORT', defaultValue: '8081', description: 'Порт для развертывания')
        choice(name: 'DEPLOY_ENV', choices: ['dev', 'staging'], description: 'Среда')
    }

    environment {
        DOCKER_IMAGE = "yourdockerhub/student-app:${BUILD_NUMBER}"
        CONTAINER_NAME = "student-app-${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Клонирование репозитория...'
                git branch: 'main',
                    url: 'https://github.com/Polisimonenko/simple-python-app.git',
                    credentialsId: 'github-credentials'
            }
        }

        stage('Setup Python') {
            steps {
                echo 'Настройка Python окружения...'
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Run Tests') {
            steps {
                echo 'Запуск тестов с генерацией JUnit XML......'
                sh '''
                    . venv/bin/activate
                    pip install unittest-xml-reporting
                    python -m xmlrunner discover -o test-results -v
                '''
            }
            post {
                success { echo '✅ Все тесты пройдены!' }
                failure { echo '❌ Тесты упали!' }
            }
        }
	stage('Publish Test Results') {
            steps {
                junit 'test-results/*.xml'
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

        stage('Push to Registry') {
            when { branch 'main' }
            steps {
                echo 'Публикация образа в Docker Hub...'
                script {
                    docker.withRegistry('', 'docker-hub-credentials') {
                        docker.image("${DOCKER_IMAGE}").push()
                        docker.image("${DOCKER_IMAGE}").push('latest')
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Развертывание контейнера...'
                script {
                    sh "docker rm -f ${CONTAINER_NAME} || true"
                    sh """
                        docker run -d \
                        --name ${CONTAINER_NAME} \
                        -p ${params.PORT}:5000 \
                        -e STUDENT_NAME='${params.STUDENT_NAME}' \
                        -e PORT=5000 \
                        ${DOCKER_IMAGE}
                    """
                    def ip = sh(script: "hostname -I | awk '{print \$1}'", returnStdout: true).trim()
                    echo "✅ Приложение доступно: http://${ip}:${params.PORT}"
                }
            }
        }
    }

    post {
        always {
            echo 'Очистка workspace...'
            cleanWs()
        }
        success { echo '🎉 Pipeline успешно выполнен!' }
        failure { echo '💥 Pipeline упал. Смотрите логи.' }
    }
}
