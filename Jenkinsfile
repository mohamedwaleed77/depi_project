pipeline {
    agent any
    triggers {
        pollSCM('0 * * * *') // every 1 hour
    }
    environment {
        TF_DIR = 'terraform' // Directory where your Terraform files are located
        ANSIBLE_PLAYBOOK_DIR = 'ansible-playbook' 
        ANSIBLE_PLAYBOOK = "${ANSIBLE_PLAYBOOK_DIR}/mainplaybook.yml"
        INVENTORY_FILE = "${ANSIBLE_PLAYBOOK_DIR}/inventory"// Ansible inventory file
        SSH_CREDENTIALS_ID = 'ssh-for-ec2'
        REPO_NAME= "ci-cd-pipeline-with-jenkins-docker-ansible-aws" //because it's very long
        REPO_URL = "https://github.com/EsraaShaabanElsayed/ci-cd-pipeline-with-jenkins-docker-ansible-aws.git"
        
        IMG_NAME="petclinic-tomcat:latest"
        DOCKER_HUB_USR="mohamedwaleed77"
        DOCKER_REPO_NAME="mohamedwaleed77/depi_petclinic:latest"
        DOCKER_HUB_TOKEN=credentials('dockerhub')
        NOTI_EMAIL= "example@gmail.com"
    }

    stages {
        stage('Checkout') {
            steps {
                    git url: "${REPO_URL}", branch: 'main' 
                    sh 'pwd ; ls -la'
            }
        }

        stage('Build') {
            steps {
                 script {
                    sh './autobuild.sh'
                    //sh "echo ${DOCKER_HUB_TOKEN} | docker login -u ${DOCKER_HUB_USR} --password-stdin" 
                    //sh "docker tag ${IMG_NAME} ${DOCKER_REPO_NAME}"
                    //sh "docker push ${DOCKER_REPO_NAME}"
                    
                
            }
        }
        }
        

        stage('Testing') {
            steps {
                sh 'docker compose up -d && sleep 60'
                dir("test_scripts") {
                    sh './tests.sh'
                }
            }
        }
         stage('Terraform Init') {
            steps {
                dir(TF_DIR) {
                    sh 'terraform init'
                }
            }
        }
        stage('Terraform Plan') {
            steps {
                dir(TF_DIR) {
                    withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-credentials', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh 'terraform plan -out=tfplan'

                    }
                }
            }
        }

        stage('Terraform Apply') {
        steps {
        dir(TF_DIR) {
        withCredentials([
                aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-credentials', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'),
                sshUserPrivateKey(credentialsId: 'aws-ec2-key-credential', keyFileVariable: 'SSH_KEY_PATH', passphraseVariable: '', usernameVariable: 'SSH_USER') 
            ])  {
                script {
                    // Apply Terraform changes
                    sh 'terraform apply -auto-approve tfplan'
                    
                    // Get the public IP of the instance
                    def instancePublicIp = sh(script: "terraform output -json | jq -r '.instance_public_ip.value'", returnStdout: true).trim()
                    echo "Instance Public IP: ${instancePublicIp}"
                    
        sh """
            pwd
            cd ../ansible-playbook/
            pwd
            touch   ${env.WORKSPACE}/ansible-playbook/inventory
            echo "[ec2]" > ${env.WORKSPACE}/ansible-playbook/inventory
            echo "${instancePublicIp}  ansible_user=${SSH_USER}  " >> ${env.WORKSPACE}/ansible-playbook/inventory
            """
    //sh "cat ${env.WORKSPACE}/ansible-playbook/inventory"
            
                }
            }
        }
    }
}


       stage('Ansible Deploy') {
            steps {
                script {
                    withCredentials([
                        aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-credentials', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        sshagent(['aws-ec2-key-credential']) {
                            dir("ansible-playbook") {
                                sh 'ansible-playbook -i inventory mainplaybook.yml'
                            }
                        }
                    }
                }
            }
        }
    

    post {
        success {
            echo 'Deployment succeeded!'
            mail to:  "${NOTI_EMAIL}",
                 subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                 body: "Good news! Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' has succeeded.\nCheck it here: ${env.BUILD_URL}"
        }

        failure {
            echo 'Deployment failed!'
            mail to:  "${NOTI_EMAIL}",
                 subject: "FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                 body: "Unfortunately, Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' has failed.\nCheck it here: ${env.BUILD_URL}"
        }
        always{
            script{
                 sh 'docker compose down'
                 sh 'rm -rf target && sudo rm -rf savedata_mw'
            }
        }
    }
}               


