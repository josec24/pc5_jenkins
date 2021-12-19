pipeline {
    agent any
    environment {
        npm_config_cache = "npm-cache"
    }
    stages {
        stage("Check versions") {
            steps {
                sh '''
                    aws --version
                    eksctl version
                    kubectl version 2>&1 | tr -d '\n'
                    docker --version
                    node --version
                    npm --version
					hadolint --version
                    ls
                '''
            }
        }
        stage("Build") {
            steps {
                sh "npm install"
            }
        }
        stage("Test") {
            steps {
                sh "CI=true npm test"
            }
        }
        stage("Release") {
            steps {
                sh "npm run build"
            }
        }
        stage("Lint") {
            steps {
                sh "hadolint infra/Dockerfile"
            }
        }
        stage("Dockerize app") {
            steps {
                sh "docker build . -f infra/Dockerfile -t josec24/jose-docker-demo:${env.BUILD_TAG}"
            }
        }
        stage("Push Docker Image") {
            steps {
                withDockerRegistry([url: "", credentialsId: "docker-credentials"]) {
                    sh "docker push josec24/jose-docker-demo:${env.BUILD_TAG}"
                }
            }
        }
        stage("Create k8s aws cluster") {
            steps{
                sh "./infra/exist-aws-k8s-cluster.sh || ./infra/create-aws-k8s-cluster.sh || exit 0"
            }
        }
        stage("Map kubectl to the k8s aws cluster and configure") {
            steps{
                withAWS(credentials: "aws-credentials", region: "us-east-1") {
                    sh "aws eks --region us-east-1 update-kubeconfig --name aws-k8s-react-app"
                    sh "kubectl config use-context arn:aws:eks:us-east-1:341033090765:cluster/jose-test-cluster"
                    sh "kubectl apply -f infra/k8s-config.yml"
                }
            }
        }
        stage("Deploy the new app dockerized") {
            steps{
                withAWS(credentials: "aws-credentials", region: "us-east-1") {
                    sh "kubectl set image deployment/aws-k8s-react-app-deployment aws-k8s-react-app=josec24/jose-docker-demo:${env.BUILD_TAG}"
                }
            }
        }
        stage("Test deployment") {
            steps{
                withAWS(credentials: "aws-credentials", region: "us-east-1") {
                    sh "kubectl get nodes"
                    sh "kubectl get deployment"
                    sh "kubectl get pod -o wide"
                    sh "kubectl get service/service-aws-k8s-react-app"
                    sh "curl \$(kubectl get service/service-aws-k8s-react-app --output jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
                }
            }
        }
        stage("Remove all unused containers, networks, images") {
            steps{
                sh "docker system prune -f"
            }
        }
    }
}





