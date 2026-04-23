pipeline {
    agent any 

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        timeout(time: 1, unit: 'HOURS')
    }

    environment {
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Load .env') {
            steps {
                withCredentials([file(credentialsId: 'app_env_file', variable: 'APP_ENV_FILE')]) {
                    sh 'cp "$APP_ENV_FILE" .env && chmod 600 .env'

                    script {
                        def props = readProperties file: '.env'

                        def requiredKeys = [
                            'DOCKER_HUB_USER',
                            'DOCKER_HUB_REPO',
                            'EC2_PUBLIC_IP',
                            'DB_HOST',
                            'DB_PORT',
                            'DB_NAME',
                            'DB_USER',
                            'DB_PASSWORD',
                            'GF_ADMIN_USER',
                            'GF_ADMIN_PASSWORD',
                            
                        ]

                        def missingKeys = requiredKeys.findAll { key -> !props[key]?.trim() }
                        if (missingKeys) {
                            error("Missing required .env values: ${missingKeys.join(', ')}")
                        }

                        props.each { key, value ->
                            env."${key}" = value?.toString()?.trim()
                        }

                        env.DOCKER_IMAGE = "${env.DOCKER_HUB_USER}/${env.DOCKER_HUB_REPO}"
                    }
                }
            }
        }

        stage('Checkout') {
            steps {
                echo 'Checking out code...'
                checkout scm
            }
        }

        stage('Install & Build') {
            agent {
                docker {
                    image 'python:3.11'
                }
            }
            steps {
                sh 'pip install -r web/requirements.txt'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    def imageName = "${DOCKER_IMAGE}"

                   
                    def appImage = docker.build("${imageName}:${IMAGE_TAG}", "-f web/Dockerfile web")

                    
                    docker.withRegistry('https://index.docker.io/v1/', 'docker-hub-credentials-id') {
                        appImage.push("${IMAGE_TAG}")
                        appImage.push("latest") 
                    }
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ec2_ssh',
                        keyFileVariable: 'ANSIBLE_SSH_KEY',
                        usernameVariable: 'ANSIBLE_SSH_USER'
                    )
                ]) {
                    sh '''
                    ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook \
                        -i "$EC2_PUBLIC_IP," \
                        -u "$ANSIBLE_SSH_USER" \
                        --private-key "$ANSIBLE_SSH_KEY" \
                        ansible/main.yml \
                        -e "web_image=${DOCKER_IMAGE}:${IMAGE_TAG}" \
                        -e "db_host=$DB_HOST" \
                        -e "db_port=$DB_PORT" \
                        -e "db_name=$DB_NAME" \
                        -e "db_user=$DB_USER" \
                        -e "db_password=$DB_PASSWORD" \
                        -e "gf_admin_user=$GF_ADMIN_USER" \
                        -e "gf_admin_password=$GF_ADMIN_PASSWORD" \
                    
                    '''
                }
            }
        }

        stage('Cleanup') {
            steps {
                sh 'docker image prune -f || true'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "Pipeline completed successfully."
        }
        failure {
            echo "Pipeline failed. Check logs for details."
        }
    }
}