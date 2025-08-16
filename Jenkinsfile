pipeline {
  agent {
    kubernetes {
      defaultContainer 'main'
      yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins-sa
  containers:
  - name: main
    image: dtzar/helm-kubectl:3.15.2
    command: ['sh','-c','sleep 999999']
    tty: true
"""
    }
  }

  stages {
    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM',
          branches: [[name: '*/main']], // change if different
          userRemoteConfigs: [[
            url: 'https://github.com/praveenkumarp893/infrastore-app.git',
            credentialsId: 'git-creds'
          ]]
        ])
        sh 'ls -la'
      }
    }

    stage('Deploy Application') {
      steps {
        withVault([
          configuration: [engineVersion: 2, skipSslVerification: true],
          vaultSecrets: [[
            path: 'secret/infrastore-app',
            secretValues: [[envVar: 'DJANGO_PASSWORD', vaultKey: 'DJANGO_SUPERUSER_PASSWORD']]
          ]]
        ]) {
          sh '''
            echo "Fetched secret from Vault: $DJANGO_PASSWORD"
            helm upgrade --install infrastore-app infrastore \
              --namespace appns --create-namespace \
              --set-string secret.DJANGO_SUPERUSER_PASSWORD="$DJANGO_PASSWORD"
          '''
        }
      }
    }
  }
}