pipeline {
  agent { label 'ubuntu' }

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  triggers { pollSCM('H/2 * * * *') }

  environment {
    WEB_ROOT   = '/var/www/html'
    BACKUP_DIR = '/var/www/backup'
  }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Validate') {
      steps {
        sh 'test -s index.html'
        sh 'grep -qi "<html" index.html'
      }
    }

    stage('Backup') {
      steps {
        sh '''
          mkdir -p $BACKUP_DIR
          if [ -f $WEB_ROOT/index.html ]; then
            cp $WEB_ROOT/index.html $BACKUP_DIR/index.html.$BUILD_NUMBER
          fi
          ls -1t $BACKUP_DIR | tail -n +6 | xargs -r -I{} rm -f $BACKUP_DIR/{}
        '''
      }
    }

    stage('Deploy') {
      steps {
        script { env.DEPLOYED = 'true' }
        sh 'cp index.html $WEB_ROOT/index.html'
      }
    }

    stage('Verify') {
      steps {
        sh '''
          code=$(curl -s -o /dev/null -w "%{http_code}" http://localhost/)
          echo "HTTP code: $code"
          [ "$code" = "200" ]
          curl -s http://localhost/ | cmp - index.html
        '''
      }
    }
  }

  post {
    success { echo 'DevOpsLabPortal deployed successfully.' }
    failure {
      script {
        if (env.DEPLOYED == 'true') {
          sh '''
            if [ -f $BACKUP_DIR/index.html.$BUILD_NUMBER ]; then
              cp $BACKUP_DIR/index.html.$BUILD_NUMBER $WEB_ROOT/index.html
              echo "ROLLBACK done from index.html.$BUILD_NUMBER"
              curl -s -o /dev/null -w "HTTP after rollback: %{http_code}\n" http://localhost/
            else
              echo "No previous version to roll back to"
            fi
          '''
        }
      }
    }
  }
}