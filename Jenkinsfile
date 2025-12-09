pipeline {
  agent any

  tools {
    maven 'maven'
  }

  environment {
    NEXUS_HOST = "192.168.11.104:8081"
    IMAGE_NAME = "tp2jenk"
    DOCKER_REPO = "192.168.11.104:5001/tp2jenk"
    GIT_URL = "https://github.com/camou92/tpjenkins-spring.git"
    K8S_DIR = "k8s" // dossier k8s dans le projet
    ARGOCD_APP_NAME = "tp2jenk"
    ARGOCD_SERVER = "192.168.39.3:32099" // ex: argocd.example.com
  }

  stages {

    stage("Clean workspace") {
      steps { deleteDir() }
    }

    stage("Clone app repo") {
      steps { sh "git clone ${GIT_URL}" }
    }

    stage("Build & Test Maven") {
      steps {
        dir("tpjenkins-spring") {
          sh "mvn -B clean install"
        }
      }
    }

    stage("Build Docker Image") {
      steps {
        dir("tpjenkins-spring") {
          sh """
            docker build -t ${IMAGE_NAME} .
            docker tag ${IMAGE_NAME} ${DOCKER_REPO}:latest
          """
        }
      }
    }

    stage("Push Docker Image to Nexus") {
      steps {
        dir("tpjenkins-spring") {
          withCredentials([usernamePassword(credentialsId: 'nexus-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            sh """
              echo $DOCKER_PASS | docker login ${DOCKER_REPO.split('/')[0]} --username $DOCKER_USER --password-stdin
              docker push ${DOCKER_REPO}:latest
              docker logout ${DOCKER_REPO.split('/')[0]}
            """
          }
        }
      }
    }

    stage("Update K8s Manifests & Push to Git") {
      steps {
        dir("tpjenkins-spring") {
          withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
            sh """
              git config user.email "cmohamed992@gmail.com"
              git config user.name "camou92"

              # Mettre à jour le tag Docker dans kustomization.yaml
              sed -i 's|newTag:.*|newTag: latest|' ${K8S_DIR}/kustomization.yaml

              # Commit & push
              git add ${K8S_DIR}/kustomization.yaml
              git commit -m "Update Docker image to latest"
              git push https://${GIT_USER}:${GIT_PASS}@${GIT_URL.replaceFirst('https://','')} HEAD:main
            """
          }
        }
      }
    }

    stage("Trigger ArgoCD Sync") {
      steps {
        withCredentials([usernamePassword(credentialsId: 'argocd-cred', usernameVariable: 'ARGO_USER', passwordVariable: 'ARGO_PASS')]) {
          sh """
            argocd login ${ARGOCD_SERVER} --username $ARGO_USER --password $ARGO_PASS --insecure
            argocd app sync ${ARGOCD_APP_NAME}
          """
        }
      }
    }

  }

  post {
    always { cleanWs() }
    success { echo "🚀 Pipeline SUCCESS — App deployed on Kubernetes via ArgoCD" }
    failure { echo "❌ Pipeline FAILED — Check logs" }
  }
}
