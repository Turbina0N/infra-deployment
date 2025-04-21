pipeline {
  agent { label 'turbina' }

  environment {
    BOT_NAME  = "${params.BOT_NAME}"
    BOT_TOKEN = "${params.BOT_TOKEN}"
  }

  stages {
    // Дефолтный checkout SCM происходит автоматически перед первым stage

    stage('Fix Terraform Permissions') {
      steps {
        echo "→ Меняем владельца и права в ${WORKSPACE}/terraform"
        sh '''
          set -e
          sudo chown -R $(id -u):$(id -g) "${WORKSPACE}/terraform"
          chmod -R u+rwX "${WORKSPACE}/terraform"
        '''
      }
    }

    stage('Terraform Init & Apply') {
      steps {
        echo "→ Запускаем Terraform в ${WORKSPACE}/terraform"
        sh '''
          set -e
          cd "${WORKSPACE}/terraform"
          terraform init -input=false
          terraform plan -lock=false -var-file=terraform.tfvars
          terraform apply -lock=false -var-file=terraform.tfvars -auto-approve
        '''
      }
    }

    stage('Extract Server IP') {
      steps {
        echo "→ Извлекаем IP через terraform output"
        script {
          env.SERVER_IP = sh(
            script: """
              cd "${WORKSPACE}/terraform"
              terraform output -raw server_ip
            """.stripIndent(),
            returnStdout: true
          ).trim()
          echo "→ Получили IP: ${env.SERVER_IP}"
        }
      }
    }
    
    stage('Add to known_hosts') {
      steps {
        echo "→ Добавляем ${env.SERVER_IP} в ~/.ssh/known_hosts"
        sh '''
          set -e
          ssh-keyscan -H "${SERVER_IP}" >> ~/.ssh/known_hosts
        '''
      }
    }



    stage('Run Ansible Playbook') {
      steps {
        echo "→ Прогоняем Ansible‑плейбук на ${env.SERVER_IP}"
        sh '''
          set -e
          cd "${WORKSPACE}/terraform"
          ansible-playbook \
            -i "${SERVER_IP}," \
            -u ubuntu \
            --private-key ~/.ssh/server \
            ansible-quizbot.yml \
            --extra-vars "bot_name=${BOT_NAME} bot_token=${BOT_TOKEN}"
        '''
      }
    }
  }

  post {
    success {
      echo 'Деплой успешно завершён'
    }
    failure {
      echo 'Произошла ошибка, см. логи'
    }
  }
}
