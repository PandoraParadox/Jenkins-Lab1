pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate') {
            steps {
                sh 'test -s index.html'
                sh 'ansible-playbook --syntax-check -i ansible/inventory.ini.example ansible/deploy-iis.yml'
            }
        }

        stage('Deploy to IIS') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'win-srv01-winrm',
                        usernameVariable: 'WINRM_USER',
                        passwordVariable: 'WINRM_PASSWORD'
                    )
                ]) {
                    sh '''
                        ansible-playbook \
                          -i ansible/inventory.ini.example \
                          ansible/deploy-iis.yml \
                          -e "ansible_user=${WINRM_USER}" \
                          -e "ansible_password=${WINRM_PASSWORD}"
                    '''
                }
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    ansible win_srv01 \
                      -i ansible/inventory.ini.example \
                      -m ansible.windows.win_uri \
                      -a "url=http://localhost:8080 return_content=true"
                '''
            }
        }
    }

    post {
        success {
            echo 'DevOpsLabPortal deployed successfully.'
        }
        failure {
            echo 'Deployment failed. Check the Ansible output and restore the IIS backup if required.'
        }
    }
}
