pipeline {
    agent any

    environment {
        APP = 'nginx'
        REGION = 'eu-west-3'
        ECRURL = 'http://209349368624.dkr.ecr.eu-west-3.amazonaws.com'
        AWS_CRED = 'ecr-access-key'
        ECRCRED = "ecr:eu-west-3:${AWS_CRED}"
        GIT_COMMIT_HASH = "${sh(script:"git log -n 1 --pretty=format:'%H'", returnStdout: true).trim()}"
        PROJECT_URL = 'http://a7f5c012a1d614fa899a2fd5d592e9dd-1112314486.eu-west-3.elb.amazonaws.com/'
        CLUSTERNAME = 'cluster-jenkins'
        RECEIPIENT_EMAIL = 'imadnajmi9@gmail.com'
        KUBECONFIG_PATH = "./kubeconfig"
    }

    stages {
        stage('Run CS Fixer & Tests') {
            steps {
                script {
                    sh 'docker-compose up --build -d'
                    sh 'docker-compose exec php composer install'
                    sh 'docker-compose exec php php bin/console doctrine:migrations:migrate -n'
                    sh 'docker-compose exec php php bin/console doctrine:fixtures:load -n'
                    sh 'docker-compose exec php ./vendor/bin/php-cs-fixer --diff --dry-run fix'
                    sh 'docker-compose exec php ./vendor/bin/phpstan analyse --memory-limit=1G'
                    sh 'docker-compose exec php php bin/phpunit'
                    sh 'docker-compose down'
                }
            }
        }

        stage('Login AWS ECR'){
            steps {
                withAWS(credentials: "${AWS_CRED}", region: 'eu-west-3') {
                    script{
                        sh("eval \$(aws ecr get-login --no-include-email | sed 's|https://||')")
                    }
                }
            }
        }

        stage('Build PHP Image'){
            steps {
                script {
                    app = docker.build("jenkins-deployment/php", '-f ./docker/php/Dockerfile .')
                }
                script {
                    docker.withRegistry(ECRURL, ECRCRED){
                        app.push("${GIT_COMMIT_HASH}")
                        app.push("develop")
                    }
                }
            }
        }

        stage('Build Nginx Image'){
            steps {
                script {
                    nginx = docker.build("jenkins-deployment/nginx", '-f ./docker/nginx/Dockerfile .')
                }
                script {
                    docker.withRegistry(ECRURL, ECRCRED){
                        nginx.push("${GIT_COMMIT_HASH}")
                        nginx.push("develop")
                    }
                }
            }
        }

       stage('EKS Deploy Develop') {
            agent {
                docker {
                    image 'alpine/k8s:1.15.11'
                }
            }
            steps {
                withAWS(credentials: "${AWS_CRED}", region: "${REGION}") {
                    sh "aws eks --region ${REGION} update-kubeconfig --name ${CLUSTERNAME} --kubeconfig ${KUBECONFIG_PATH}"
                    sh "sed -i 's/TAG/${GIT_COMMIT_HASH}/g' ./kubernetes/develop/deploy.yaml"
                    sh 'kubectl apply -f ./kubernetes/develop/ -R --kubeconfig ./kubeconfig'
                }
            }
        }

        stage('Validate Deployment') {
            agent {
                docker { image 'alpine/k8s:1.15.11' }
            }
            steps {
                withAWS(credentials: "${AWS_CRED}", region: "${REGION}") {
                    sh "aws eks --region ${REGION} update-kubeconfig --name ${CLUSTERNAME} --kubeconfig ${KUBECONFIG_PATH}"
                    sh '''
                        POD_NAME=$(kubectl get pods --kubeconfig ${KUBECONFIG_PATH} | grep ${APP}-deployment | grep "ContainerCreating" | awk '{print $1}' | head -n 1)
                        if ! kubectl rollout status deployment ${APP}-deployment --timeout=600s --kubeconfig ${KUBECONFIG_PATH}; then
                            kubectl logs --kubeconfig ${KUBECONFIG_PATH} --tail=10 $POD_NAME application
                            kubectl rollout undo deployment ${APP}-deployment --kubeconfig ${KUBECONFIG_PATH}
                            kubectl rollout status deployment ${APP}-deployment --kubeconfig ${KUBECONFIG_PATH} --timeout=600s
                            exit 1
                        fi
                    '''
                }
            }
        }
    }

   post {
        failure {
            emailext subject: 'Pipeline Failed',
            body: 'There was a failure in the pipeline. Please check the build logs for more details.',
            to: "${RECEIPIENT_EMAIL}",
            attachLog: true
        }

        success {
            emailext subject: 'Pipeline Successful',
            body: "The pipeline was successful, see ${PROJECT_URL}. Congratulations!",
            to: "${RECEIPIENT_EMAIL}"
        }

        always {
            cleanWs()
        }
    }
}
