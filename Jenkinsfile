pipeline {
  agent any

  tools {
    maven 'maven'
  }

  environment {
    NEXUS_HOST = "192.168.11.104:8081"
    IMAGE_NAME = "tp2jenk"
    DOCKER_REPO = "192.168.11.104:5001/tp2jenk"
  }

  stages {

    stage("Clean up"){
      steps {
        deleteDir()
      }
    }

    stage("Clone repo"){
      steps {
        sh "git clone https://github.com/camou92/tpjenkins-spring.git"
      }
    }

    stage("Build & Test Maven"){
      steps {
        dir("tpjenkins-spring") {
          sh "mvn -B clean install"
        }
      }
    }

    stage("Prepare settings.xml for Nexus Deploy"){
      steps {
        dir("tpjenkins-spring") {
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
    }

    stage("Deploy Maven Artifacts to Nexus"){
      steps {
        dir("tpjenkins-spring") {
          sh "mvn -B deploy -s settings.xml -DskipTests=false"
        }
      }
    }

    stage("Build Docker Image"){
      steps {
        dir("tpjenkins-spring") {
          sh "docker build -t ${IMAGE_NAME} ."
        }
      }
    }

    stage("Push Docker Image to Nexus"){
      steps {
        dir("tpjenkins-spring") {
          withCredentials([usernamePassword(credentialsId: 'nexus-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            sh """
              docker tag ${IMAGE_NAME} ${DOCKER_REPO}:latest

              echo ${DOCKER_PASS} | docker login ${DOCKER_REPO.split('/')[0]} --username ${DOCKER_USER} --password-stdin

              docker push ${DOCKER_REPO}:latest

              docker logout ${DOCKER_REPO.split('/')[0]}
            """
          }
        }
      }
    }

    stage("Deploy & Run Application with Docker Compose"){
      steps {
        dir("tpjenkins-spring") {

          sh """
            # Stop running application (if exists)
            docker compose down || true

            # Pull the latest image from Nexus
            docker pull ${DOCKER_REPO}:latest

            # Start application
            docker compose up -d
          """
        }
      }
    }

  } // end stages

  post {
    always {
      cleanWs()
    }
    success {
      echo "🚀 Pipeline SUCCESS — Application is deployed and running."
    }
    failure {
      echo "❌ Pipeline FAILED — Check logs."
    }
  }

}
