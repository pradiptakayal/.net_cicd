pipeline {
    agent any
    environment {
      SONAR_TOKEN = credentials('SONAR_TOKEN')  // Jenkins credentials ID for SonarQube token
      PATH = "$PATH:$HOME/.dotnet/tools"  // Ensure .NET tools are in PATH
    }
    
    stages {

        stage('CLEAN WORKSPACE') {
            steps {
                cleanWs()
            }
        }

        stage('Check PATH') {
            steps {
                sh 'echo $PATH'
            }
        }

     stage('CODE CHECKOUT') {
            steps {
                git 'https://github.com/pradiptakayal/web.git'
            }
        }

         stage('MODIFIED IMAGE TAG') {
            steps {
                sh '''
                   sed "s/image-name:latest/$JOB_NAME:v1.$BUILD_ID/g" playbooks/dep_svc.yml
                   sed -i "s/image-name:latest/$JOB_NAME:v1.$BUILD_ID/g" playbooks/dep_svc.yml
                     sed -i "s/IMAGE_NAME/$JOB_NAME:v1.$BUILD_ID/g" webapp/Views/Home/Index.cshtml
                   '''
            }
        }

         stage('Restore') {
            steps {
                dir('/var/lib/jenkins/workspace/project-2/webapp/') { // Ensure this is the correct path to your project directory
                    sh 'dotnet restore'
                }
            }
        }
        
        stage('Build') {
            steps {
                dir('/var/lib/jenkins/workspace/project-2/webapp') {
                    sh 'dotnet build --configuration Release'
                }
            }
        }


       
        stage('SonarQube Analysis') {
            steps {
                dir('webapp') {  // Adjusted to your project directory
                    sh '''
                        dotnet sonarscanner begin /k:"project-2" \
                            /d:sonar.host.url="http://192.168.1.19:9000" \
                            /d:sonar.login="${SONAR_TOKEN}"
                        
                        dotnet build --configuration Release
                        
                        dotnet sonarscanner end /d:sonar.login="${SONAR_TOKEN}"
                    '''
                }
            }
        }

         stage('COPY Project & DOCKERFILE') {
            steps {
                sh 'ansible-playbook playbooks/create_directory.yml'
            }
        }

        stage('PUSH IMAGE ON DOCKERHUB') {
            environment {
            dockerhub_user = credentials('DOCKERHUB_USER')            
            dockerhub_pass = credentials('DOCKERHUB_PASS')
            }    
            steps {
                sh 'ansible-playbook playbooks/push_dockerhub.yml \
                    --extra-vars "JOB_NAME=$JOB_NAME" \
                    --extra-vars "BUILD_ID=$BUILD_ID" \
                    --extra-vars "dockerhub_user=$dockerhub_user" \
                    --extra-vars "dockerhub_pass=$dockerhub_pass"'              
            }
        }

        
        stage('DEPLOYMENT ON EKS') {
            steps {
                sh 'ansible-playbook playbooks/create_pod_on_eks.yml \
                    --extra-vars "JOB_NAME=$JOB_NAME"'
            }            
        }   

        stage('Install Prometheus and Grafana') {
            steps {
                sh '''
                    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
                    helm repo add grafana https://grafana.github.io/helm-charts
                    helm repo update
                    helm install prometheus prometheus-community/kube-prometheus-stack
                    helm install grafana grafana/grafana
                '''
            }
        }

        stage('Configure Prometheus') {
            steps {
                sh '''
                    # Apply your Prometheus scrape config
                    kubectl apply -f playbooks/prometheus-scrape-config.yml
                '''
            }
        }

        stage('Configure Grafana') {
            steps {
                sh '''
                    # Add data source and dashboards
                    kubectl apply -f playbooks/grafana-datasource-config.yml
                    kubectl apply -f playbooks/grafana-dashboard-config.yml
                '''
            }
        }

        stage('Verify Monitoring Setup') {
            steps {
                script {
                    def prometheusUrl = sh(script: "kubectl get svc prometheus-kube-prometheus-prometheus -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'", returnStdout: true).trim()
                    def grafanaUrl = sh(script: "kubectl get svc grafana -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'", returnStdout: true).trim()
                    echo "Prometheus is accessible at: http://$prometheusUrl"
                    echo "Grafana is accessible at: http://$grafanaUrl"
                }
            }
        }
    }
}
