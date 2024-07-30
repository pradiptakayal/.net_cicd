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


         stage('COPY Project & DOCKERFILE') {
            steps {
                sh 'ansible-playbook playbooks/create_directory.yml'
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
