pipeline {
    agent any

    parameters {
        choice(
            name: 'ACTION',
            choices: ['deploy', 'rollback', 'build', 'push'],
            description: 'Pipeline action to execute'
        )
        string(
            name: 'VERSION',
            defaultValue: '',
            description: 'Target version tag (e.g. v1.0.0). Required for rollback or manual runs.'
        )
    }

    environment {
        AWS_REGION     = 'ap-southeast-2'
        AWS_ACCOUNT_ID = '251528148716'
        ECR_REPO_NAME  = 'my-app-h2'
        ECR_REGISTRY   = "\.dkr.ecr.\.amazonaws.com"
        MANAGER_IP     = '32.236.227.255'
        SERVICE_NAME   = 'my-stack_web'
        DEPLOY_VERSION = ''
    }

    stages {
        stage('Resolve Target Version') {
            steps {
                script {
                    if (env.TAG_NAME) {
                        env.DEPLOY_VERSION = env.TAG_NAME
                        echo "Triggered by Git tag push: \"
                    } else if (params.VERSION?.trim()) {
                        env.DEPLOY_VERSION = params.VERSION.trim()
                        echo "Version set via parameters: \"
                    } else {
                        error("Deployment halted: No VERSION provided and not triggered by a Git tag.")
                    }
                }
            }
        }

        stage('Checkout Source') {
            when {
                expression { params.ACTION != 'rollback' }
            }
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            when {
                expression { params.ACTION in ['build', 'deploy'] }
            }
            steps {
                sh "docker build -t \/\:\ ."
            }
        }

        stage('Push to Amazon ECR') {
            when {
                expression { params.ACTION in ['push', 'deploy'] }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-ecr-credentials',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh """
                        docker run --rm \
                          -e AWS_ACCESS_KEY_ID="\" \
                          -e AWS_SECRET_ACCESS_KEY="\" \
                          -e AWS_DEFAULT_REGION="\" \
                          amazon/aws-cli ecr get-login-password --region "\" | docker login --username AWS --password-stdin "\"
                        docker push "\/\:\"
                    """
                }
            }
        }

        stage('Verify Image in ECR (Rollback Guard)') {
            when {
                expression { params.ACTION == 'rollback' }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-ecr-credentials',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh """
                        echo "Verifying image tag \ exists in Amazon ECR..."
                        docker run --rm \
                          -e AWS_ACCESS_KEY_ID="\" \
                          -e AWS_SECRET_ACCESS_KEY="\" \
                          -e AWS_DEFAULT_REGION="\" \
                          amazon/aws-cli ecr describe-images --repository-name \ --image-ids imageTag=\ --region \ > /dev/null
                    """
                }
            }
        }

        stage('Deploy / Rollback on Swarm') {
            when {
                expression { params.ACTION in ['deploy', 'rollback'] }
            }
            steps {
                sshagent(['swarm-manager-ssh']) {
                    sh """
                        echo "Executing service update to image tag: \ on Docker Swarm..."
                        ssh -o StrictHostKeyChecking=no ubuntu@\ "
                            aws ecr get-login-password --region \ | docker login --username AWS --password-stdin \
                            docker service update --image \/\:\ --with-registry-auth \
                        "
                    """
                }
            }
        }
    }
}
