pipeline {
  agent any

  tools {
    maven 'maven'
  }

  environment {
    NEXUS_HOST = "192.168.122.1:8081"
    IMAGE_NAME = "tp2jenk"
    DOCKER_REPO = "192.168.122.1:5001/tp2jenk"
    GIT_APP_REPO = "https://github.com/camou92/tpjenkins-spring.git"
    K8S_DIR = "k8s"
    ARGOCD_APP_NAME = "tp2jenk"
    ARGOCD_SERVER = "192.168.39.3:32099"
    DOCKER_IMAGE = "${DOCKER_REPO}:${BUILD_NUMBER}"
  }

  stages {

    stage("Clean workspace safe") {
      steps {
        cleanWs()
      }
    }

    stage('0. Checkout App Repo') {
      steps {
        checkout([$class: 'GitSCM', branches: [[name: '*/main']],
            userRemoteConfigs: [[url: "${GIT_APP_REPO}"]]])
      }
    }

    stage('1. Build & Tests') {
      steps {
        sh "mvn -B clean install"
      }
    }

    stage("Prepare Maven settings.xml for Nexus") {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-cred', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
          sh '''
cat > settings.xml <<EOF
<settings>
  <servers>
    <server>
      <id>nexus</id>
      <username>${NEXUS_USER}</username>
      <password>${NEXUS_PASS}</password>
    </server>
  </servers>
</settings>
EOF
          '''
        }
      }
    }

    stage("Deploy Maven Artifacts") {
      steps {
        sh "mvn deploy -s settings.xml -DskipTests=false"
      }
    }

    stage('2. Build & Push Docker') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-cred',
          usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {

          sh '''
            set -e
            # Build Docker image avec tag BUILD_NUMBER
            docker build -t ${DOCKER_IMAGE} .

            # Extraire l'hôte du repo Docker pour login
            DOCKER_HOST=$(echo ${DOCKER_REPO} | cut -d/ -f1)

            echo "$DOCKER_PASS" | docker login $DOCKER_HOST --username "$DOCKER_USER" --password-stdin

            # Push uniquement l'image versionnée
            docker push ${DOCKER_IMAGE}

            docker logout $DOCKER_HOST
          '''
        }
      }
    }

    stage('3. Update Kubernetes Manifests') {
  steps {
    sh '''
      set -e

      # Remplacer BUILD_NUMBER_PLACEHOLDER dans kustomization.yaml
      sed -i "s|BUILD_NUMBER_PLACEHOLDER|${BUILD_NUMBER}|" ${K8S_DIR}/kustomization.yaml

      git config user.email "cmohamed992@gmail.com"
      git config user.name "camou92"

      # Créer ou se placer sur main
      git checkout -B main

      git add ${K8S_DIR}/kustomization.yaml
      git diff --cached --quiet || git commit -m "Update image to ${BUILD_NUMBER}"

      git push origin main
    '''
  }
}


    stage("4. Trigger ArgoCD Sync") {
      steps {
        withCredentials([usernamePassword(credentialsId: 'argocd-cred',
          usernameVariable: 'ARGO_USER', passwordVariable: 'ARGO_PASS')]) {

          sh '''
            argocd login ${ARGOCD_SERVER} --username "$ARGO_USER" --password "$ARGO_PASS" --insecure
            argocd app sync ${ARGOCD_APP_NAME}
          '''
        }
      }
    }
  }

  post {
    success {
      slackSend(
        channel: '#tous-camoutech',
        color: '#36a64f',
        message: "🎉 SUCCESS — Build ${BUILD_NUMBER} déployé 🚀"
      )
    }
    failure {
      slackSend(
        channel: '#tous-camoutech',
        color: '#ff0000',
        message: "❌ Pipeline ${BUILD_NUMBER} échoué ⚠️"
      )
    }
  }
}
