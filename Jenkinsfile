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
    GIT_K8S_REPO = "https://github.com/camou92/tpjenk-k8s.git"
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

    stage("Prepare Maven settings.xml for Nexus"){
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

    stage("Deploy Maven Artifacts"){
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
            docker build -t ${DOCKER_IMAGE} .
            docker tag ${DOCKER_IMAGE} ${DOCKER_REPO}:latest
            
            echo "$DOCKER_PASS" | docker login ${DOCKER_REPO.split('/')[0]} \
              --username "$DOCKER_USER" --password-stdin

            docker push ${DOCKER_IMAGE}
            docker push ${DOCKER_REPO}:latest
            docker logout ${DOCKER_REPO.split('/')[0]}
          '''
        }
      }
    }

    stage('3. Update Kubernetes Manifests Repo') {
      steps {
        withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
          sh '''
            set -e
            rm -rf k8s-repo
            # Clone avec token sécurisé
            git clone https://${GITHUB_TOKEN}@github.com/camou92/tpjenk-k8s.git k8s-repo
            cd k8s-repo

            git config user.email "cmohamed992@gmail.com"
            git config user.name "camou92"

            # Mettre à jour le BUILD_NUMBER dans kustomization.yaml
            sed -i "s|BUILD_NUMBER_PLACEHOLDER|${BUILD_NUMBER}|" ${K8S_DIR}/kustomization.yaml

            git add .
            git diff --cached --quiet || git commit -m "Update image to ${BUILD_NUMBER}"

            # Push sécurisé avec token
            git push https://${GITHUB_TOKEN}@github.com/camou92/tpjenk-k8s.git HEAD:main
          '''
        }
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
