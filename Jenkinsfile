pipeline {
    agent any

    environment {
        SS_KEY = credentials('ec2-key')
        TF_DIR = 'terraform' // Directory where your Terraform files are located
        ANSIBLE_PLAYBOOK_DIR = 'ansible-playbook' 
        ANSIBLE_PLAYBOOK = '${ANSIBLE_PLAYBOOK_DIR}/mainplaybook.yml' 
        INVENTORY_FILE = '${ANSIBLE_PLAYBOOK_DIR}/inventory' // Ansible inventory file
       
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'test-terraform', url: 'https://github.com/EsraaShaabanElsayed/ci-cd-pipeline-with-jenkins-docker-ansible-aws.git'
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
                    withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-credentials', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        script {
                            sh 'terraform apply -auto-approve tfplan '
                            
                        def instancePublicIp = sh(script: "terraform output -json | jq -r '.instance_public_ip.value'", returnStdout: true).trim()
                            
                            echo "Instance Public IP: ${instancePublicIp}"
                            echo "Writing to inventory file with the following values:"
                            echo "[ec2]"
                            echo "${instancePublicIp} ansible_ssh_private_key_file=${SS_KEY} ansible_user=ubuntu"

                            writeFile file: INVENTORY_FILE, text: "[ec2]\n${instancePublicIp} ansible_ssh_private_key_file=${SS_KEY} ansible_user=ubuntu\n"
                            sh "cat ${INVENTORY_FILE}"  
                }
                           
                        }
                    }
                }
            }
        }

        stage('Ansible Deploy') {
            steps {
                script {
                    withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-credentials', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
        
                 
                    sh "ls -al"
                    sh "cat ${INVENTORY_FILE}"
                    echo "SSH Key: ${SS_KEY}"

                    // Run the Ansible playbook
                    sh "ansible-playbook -i ${INVENTORY_FILE} ${ANSIBLE_PLAYBOOK} -vvv"
        
                    
                }
                }
            }
        }
    }
}
