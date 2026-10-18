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
        ECR_REGISTRY   = '251528148716.dkr.ecr.ap-southeast-2.amazonaws.com'
        MANAGER_IP     = '32.236.227.255'
        SERVICE_NAME   = 'my-stack_web'
    }

    stages {
        stage('Resolve Target Version') {
            steps {
                script {
                    def targetVer = ''
                    if (env.TAG_NAME) {
                        targetVer = env.TAG_NAME
                    } else if (params.VERSION?.trim()) {
                        targetVer = params.VERSION.trim()
                    } else {
                        // Fallback: extract latest git tag from workspace checkout
                        targetVer = sh(script: 'git describe --tags --always', returnStdout: true).trim()
                    }

                    if (!targetVer) {
                        error("Deployment halted: No target version found.")
                    }

                    env.DEPLOY_VERSION = targetVer
                    echo "Deploying version: ${env.DEPLOY_VERSION} with action: ${params.ACTION}"
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
                sh "docker build -t ${env.ECR_REGISTRY}/${env.ECR_REPO_NAME}:${env.DEPLOY_VERSION} ."
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
                          -e AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID}" \
                          -e AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY}" \
                          -e AWS_DEFAULT_REGION="${env.AWS_REGION}" \
                          amazon/aws-cli ecr get-login-password --region "${env.AWS_REGION}" | docker login --username AWS --password-stdin "${env.ECR_REGISTRY}"
                        docker push "${env.ECR_REGISTRY}/${env.ECR_REPO_NAME}:${env.DEPLOY_VERSION}"
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
                        echo "Verifying image tag ${env.DEPLOY_VERSION} exists in Amazon ECR..."
                        docker run --rm \
                          -e AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID}" \
                          -e AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY}" \
                          -e AWS_DEFAULT_REGION="${env.AWS_REGION}" \
                          amazon/aws-cli ecr describe-images --repository-name ${env.ECR_REPO_NAME} --image-ids imageTag=${env.DEPLOY_VERSION} --region ${env.AWS_REGION} > /dev/null
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
                        echo "Executing service update to image tag: ${env.DEPLOY_VERSION} on Docker Swarm..."
                        ssh -o StrictHostKeyChecking=no ubuntu@${env.MANAGER_IP} "
                            aws ecr get-login-password --region ${env.AWS_REGION} | docker login --username AWS --password-stdin ${env.ECR_REGISTRY}
                            docker service update --image ${env.ECR_REGISTRY}/${env.ECR_REPO_NAME}:${env.DEPLOY_VERSION} --with-registry-auth ${env.SERVICE_NAME}
                        "
                    """
                }
            }
        }
    }
}
