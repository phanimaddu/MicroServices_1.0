
pipeline {

    agent any

    parameters {
        string(
            name: 'IMAGE_TAG',
            defaultValue: '2.0',
            description: 'Docker image version'
        )

        choice(
            name: 'ACTION',
            choices: [
                'BUILD_AND_DEPLOY',
                'BUILD_ONLY',
                'DEPLOY_ONLY'
            ],
            description: 'Pipeline action'
        )
    }

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '658606186375'

        EC2_HOST = '172.31.5.188'

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        REMOTE_DIR = '/home/ubuntu/microservices/docker/nginx'
    }

    stages {

        // CHECKOUT

        stage('Checkout') {
            steps {
                git branch: 'feature/docker',
                    credentialsId: 'github-credentials',
                    url: 'https://github.com/phanimaddu/MicroServices_1.0.git'
            }
        }

        // VALIDATE

        stage('Validate') {
            steps {
                sh '''
                    set -e

                    echo "Validating Docker and Node..."

                    node --version
                    npm --version
                    docker --version
                    docker compose version

                    if [ ! -f docker-compose.yml ]; then
                        echo "ERROR: docker-compose.yml not found"
                        exit 1
                    fi

                    npm ci --omit=dev

                    ECR_REGISTRY="$ECR_REGISTRY" \
                    IMAGE_TAG="$IMAGE_TAG" \
                    docker compose config --quiet

                    echo "Validation completed."
                '''
            }
        }

        // BUILD DOCKER IMAGES

        stage('Build Images') {
            when {
                expression {
                    params.ACTION == 'BUILD_AND_DEPLOY' ||
                    params.ACTION == 'BUILD_ONLY'
                }
            }

            steps {
                script {

                    def services = [
                        [name: 'user-service', file: 'Dockerfile.user'],
                        [name: 'product-service', file: 'Dockerfile.product'],
                        [name: 'order-service', file: 'Dockerfile.order']
                    ]

                    services.each { service ->

                        sh """
                            set -e

                            docker build \
                                -f ${service.file} \
                                -t ${ECR_REGISTRY}/${service.name}:${params.IMAGE_TAG} .
                        """
                    }
                }
            }
        }

        // ECR LOGIN

        stage('ECR Login') {
            when {
                expression {
                    params.ACTION == 'BUILD_AND_DEPLOY' ||
                    params.ACTION == 'BUILD_ONLY'
                }
            }

            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-jenkins'
                ]]) {

                    sh '''
                        set -e

                        aws ecr get-login-password \
                            --region "$AWS_REGION" \
                        | docker login \
                            --username AWS \
                            --password-stdin "$ECR_REGISTRY"
                    '''
                }
            }
        }

        // PUSH IMAGES TO ECR

        stage('Push Images') {
            when {
                expression {
                    params.ACTION == 'BUILD_AND_DEPLOY' ||
                    params.ACTION == 'BUILD_ONLY'
                }
            }

            steps {
                sh '''
                    set -e

                    docker push "$ECR_REGISTRY/user-service:$IMAGE_TAG"

                    docker push "$ECR_REGISTRY/product-service:$IMAGE_TAG"

                    docker push "$ECR_REGISTRY/order-service:$IMAGE_TAG"

                    echo "All Docker images pushed to ECR."
                '''
            }
        }

        // DEPLOY TO EC2

        stage('Deploy to EC2') {
            when {
                expression {
                    params.ACTION == 'BUILD_AND_DEPLOY' ||
                    params.ACTION == 'DEPLOY_ONLY'
                }
            }

            steps {

                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ec2-ssh-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "Preparing EC2 deployment..."

                        chmod 600 "$SSH_KEY"

                        SSH_OPTIONS="-i $SSH_KEY -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=15"

                        echo "Creating deployment directory..."

                        ssh $SSH_OPTIONS "$SSH_USER@$EC2_HOST" \
                            "mkdir -p $REMOTE_DIR"

                        echo "Copying Docker Compose file..."

                        scp $SSH_OPTIONS \
                            docker-compose.yml \
                            "$SSH_USER@$EC2_HOST:$REMOTE_DIR/docker-compose.yml"

                        echo "Deploying containers on EC2..."

                        ssh $SSH_OPTIONS "$SSH_USER@$EC2_HOST" \
                            "AWS_REGION='$AWS_REGION' ECR_REGISTRY='$ECR_REGISTRY' IMAGE_TAG='$IMAGE_TAG' REMOTE_DIR='$REMOTE_DIR' bash -s" <<'REMOTE'

                        set -e

                        cd "$REMOTE_DIR"

                        echo "Logging into AWS ECR..."

                        aws ecr get-login-password \
                            --region "$AWS_REGION" \
                        | docker login \
                            --username AWS \
                            --password-stdin "$ECR_REGISTRY"

                        echo "Pulling latest images..."

                        docker compose pull

                        echo "Starting microservices..."

                        docker compose up -d --remove-orphans

                        echo "Checking running containers..."

                        docker compose ps

                        echo "EC2 deployment completed successfully."

                        REMOTE
                    '''
                }
            }
        }

        // DEPLOYMENT VERIFICATION

        stage('Deployment Verification') {

            when {
                expression {
                    params.ACTION == 'BUILD_AND_DEPLOY' ||
                    params.ACTION == 'DEPLOY_ONLY'
                }
            }

            steps {

                sh '''
                    set -e

                    echo "Waiting for microservices..."

                    sleep 15

                    echo "Checking User Service..."

                    curl --fail --silent --show-error \
                        --retry 5 \
                        --retry-delay 5 \
                        --retry-connrefused \
                        "http://$EC2_HOST:8081/users"

                    echo "Checking Product Service..."

                    curl --fail --silent --show-error \
                        --retry 5 \
                        --retry-delay 5 \
                        --retry-connrefused \
                        "http://$EC2_HOST:8081/products"

                    echo "Checking Order Service..."

                    curl --fail --silent --show-error \
                        --retry 5 \
                        --retry-delay 5 \
                        --retry-connrefused \
                        "http://$EC2_HOST:8081/orders"

                    echo "All microservices are responding successfully!"
                '''
            }
        }
    }

    // POST ACTIONS

    post {

        success {
            echo 'Microservices pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check Jenkins console output.'
        }

        always {
            echo 'Pipeline execution completed.'
        }
    }
}
