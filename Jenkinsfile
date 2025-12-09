pipeline {

  ////////////////////////////////////////////////////////////////////////////
  // AGENT DOCKER POUR MAVEN + DOCKER BUILDS
  ////////////////////////////////////////////////////////////////////////////
  agent {
    docker {
      image 'donald284/maven-jenkins-agent:v3'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
    }
  }

  ////////////////////////////////////////////////////////////////////////////
  // VARIABLES GLOBALES
  ////////////////////////////////////////////////////////////////////////////
  environment {
    NEXUS_HOST = "192.168.11.104:8081"
    IMAGE_NAME = "tp2jenk"
    DOCKER_REPO = "192.168.11.104:5001/tp2jenk"
    GIT_URL = "https://github.com/camou92/tpjenkins-spring.git"
    K8S_DIR = "k8s" // dossier k8s dans le projet
    ARGOCD_APP_NAME = "tp2jenk"
    ARGOCD_SERVER = "192.168.39.3:32099"
    DOCKER_IMAGE = "${DOCKER_REPO}:${BUILD_NUMBER}" // version Docker basée sur BUILD_NUMBER
  }

  stages {

    /////////////////////////////////////////////////////////
    // 0. CHECKOUT GIT
    /////////////////////////////////////////////////////////
    stage('0. Checkout') {
      steps {
        git branch: 'main', url: "${GIT_URL}"
      }
    }

    /////////////////////////////////////////////////////////
    // 1. MAVEN BUILD
    /////////////////////////////////////////////////////////
    stage('1. Build et Tests (Maven)') {
      steps {
        dir("tpjenkins-spring") {
          sh "mvn -B clean install"
        }
      }
    }

    /////////////////////////////////////////////////////////
    // 2. BUILD & PUSH DOCKER IMAGE SUR NEXUS
    /////////////////////////////////////////////////////////
    stage('2. Build et Push Docker') {
      steps {
        dir("tpjenkins-spring") {
          withCredentials([usernamePassword(credentialsId: 'nexus-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            sh """
              docker build -t ${DOCKER_IMAGE} .
              docker tag ${DOCKER_IMAGE} ${DOCKER_REPO}:latest

              echo $DOCKER_PASS | docker login ${DOCKER_REPO.split('/')[0]} --username $DOCKER_USER --password-stdin
              docker push ${DOCKER_IMAGE}
              docker push ${DOCKER_REPO}:latest
              docker logout ${DOCKER_REPO.split('/')[0]}
            """
          }
        }
      }
    }

    /////////////////////////////////////////////////////////
    // 3. UPDATE K8S MANIFESTS (TAG DOCKER) ET PUSH GIT
    /////////////////////////////////////////////////////////
    stage('3. Mise à jour des manifests Kubernetes') {
      agent none
      steps {
        withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
          sh """
            git clone ${GIT_URL} temp-k8s-repo
            cd temp-k8s-repo

            git config user.email "cmohamed992@gmail.com"
            git config user.name "camou92"

            # Mettre à jour le tag Docker dans kustomization.yaml
            sed -i "s|newTag:.*|newTag: ${BUILD_NUMBER}|" ${K8S_DIR}/kustomization.yaml

            git add ${K8S_DIR}/kustomization.yaml
            git diff --cached --quiet || git commit -m "Update Docker image to ${BUILD_NUMBER}"

            git push https://${GITHUB_TOKEN}@github.com/camou92/tpjenkins-spring.git HEAD:main
          """
        }
      }
    }
}

    /////////////////////////////////////////////////////////
    // 4. TRIGGER ARGOCD SYNC
    /////////////////////////////////////////////////////////
    stage("4. Trigger ArgoCD Sync") {
      steps {
        withCredentials([usernamePassword(credentialsId: 'argocd-cred', usernameVariable: 'ARGO_USER', passwordVariable: 'ARGO_PASS')]) {
          sh """
            argocd login ${ARGOCD_SERVER} --username $ARGO_USER --password $ARGO_PASS --insecure
            argocd app sync ${ARGOCD_APP_NAME}
          """
        }
      }
    }

  } // end stages

  ////////////////////////////////////////////////////////////////////////////
  // NOTIFICATIONS SLACK
  ////////////////////////////////////////////////////////////////////////////
  post {

    success {
      slackSend(
        channel: '#tous-camoutech',
        color: '#36a64f',
        message: "🎉 SUCCESS — Build #${BUILD_NUMBER} déployé avec succès sur Kubernetes via ArgoCD ! 🚀"
      )
    }

    failure {
      slackSend(
        channel: '#tous-camoutech',
        color: '#ff0000',
        message: "❌ FAILURE — Le pipeline #${BUILD_NUMBER} a échoué ! ⚠️"
      )
    }

  } // end post

} // end pipeline
