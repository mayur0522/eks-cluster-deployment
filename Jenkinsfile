pipeline {
    agent any

    parameters {
        choice(
            name: 'ACTION',
            choices: ['apply', 'destroy'],
            description: 'Select the action to perform'
        )
    }

    environment {
        AWS_REGION = 'ap-south-1'
        EKS_CLUSTER = 'ankit-cluster'
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/ankit-jagtap-devops/terraform-eks-nodegroup.git'
            }
        }

        stage("Terraform Init") {
            steps {
                sh "terraform init -reconfigure"
            }
        }

        stage("Terraform Plan") {
            steps {
                sh "terraform plan"
            }
        }

        stage("Terraform Action") {
            steps {
                script {
                    if (params.ACTION == 'apply') {
                        echo 'Applying infrastructure...'
                        sh 'terraform apply --auto-approve'
                    } else if (params.ACTION == 'destroy') {
                        echo 'Destroying infrastructure...'
                        sh 'terraform destroy --auto-approve'
                    } else {
                        error("Unknown ACTION: ${params.ACTION}")
                    }
                }
            }
        }

        stage("Configure Kubeconfig") {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                echo "Setting up kubeconfig for EKS access"
                sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}"
            }
        }

        stage("Install Observability Stack") {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                sh '''
                # Prometheus & Grafana
                helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
                helm repo update
                helm install prometheus prometheus-community/kube-prometheus-stack || echo "Prometheus already installed"

                # OpenObserve
                helm repo add openobserve https://charts.openobserve.ai
                helm repo update
                helm install openobserve openobserve/openobserve || echo "OpenObserve already installed"

                # Expose OpenObserve
                kubectl expose deployment openobserve \
                  --type=LoadBalancer \
                  --name=openobserve-service \
                  --port=80 \
                  --target-port=5080 || echo "Already exposed"
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed. Check logs above."
        }
    }
}
