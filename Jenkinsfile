pipeline {
    agent any
    environment {
      PATH = "$PATH:/usr/share/dotnet:$HOME/.dotnet/tools"
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


        stage('Publish') {
            steps {
                dir('/var/lib/jenkins/workspace/project-2/webapp') {
                    sh 'dotnet publish --configuration Release --output ./publish'
                }
            }
        }


        


        stage('Sonar Scanner') {
            environment {
                sonar_token = credentials('SONAR_TOKEN1')
            }
            steps {
                dir('/var/lib/jenkins/workspace/project-2/webapp') {
                    sh '''
                    export PATH=$PATH:/home/vagrant/.dotnet/tools
                    dotnet sonarscanner begin /k:"project-2" /d:sonar.host.url="http://192.168.33.11:9000" /d:sonar.login="$sonar_token"
                    dotnet build
                    dotnet sonarscanner end /d:sonar.login="$sonar_token"
                    '''
                }
            }
        }

        

         stage('COPY JAR & DOCKERFILE') {
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
    }
}
