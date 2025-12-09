pipeline {
  agent any

  tools {
    maven 'maven'   // nom du Maven tool configuré dans Jenkins (Global Tool Configuration)
    // optionally docker tool if configured
  }

  environment {
    NEXUS_HOST = "192.168.11.104:8081"            // adapte si nécessaire
    IMAGE_NAME = "tp2jenk"                   // nom local de l'image
    DOCKER_REPO = "192.168.11.104:5001/tp2jenk"   // si tu utilises Nexus Docker hosted sur le port 5001
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

    stage("Build & Test (Maven)"){
      steps {
        dir("tpjenkins-spring") {
          sh "mvn -B clean install"
        }
      }
    }

    stage("Prepare Maven settings for Nexus"){
      steps {
        dir("tpjenkins-spring") {
          // on récupère les credentials Jenkins (ID 'nexus-cred') et on génère settings.xml
          withCredentials([usernamePassword(credentialsId: 'nexus-cred', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
            sh '''
              cat > settings.xml <<EOF
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                              http://maven.apache.org/xsd/settings-1.0.0.xsd">
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

    stage("Deploy to Nexus (mvn deploy)"){
      steps {
        dir("tpjenkins-spring") {
          // On utilise le settings.xml précédent
          sh "mvn -B deploy -s settings.xml -DskipTests=false"
        }
      }
    }

    stage("Build Docker image"){
      steps {
        dir("tpjenkins-spring") {
          sh "docker build -t ${IMAGE_NAME} ."
        }
      }
    }

    stage("Push Docker image to Nexus (optional)"){
      steps {
        dir("tpjenkins-spring") {
          // se connecter au registre Nexus (si tu as créé un docker hosted repo à port 5001)
          // utilise les mêmes creds 'nexus-cred' ou un autre id 'nexus-docker-cred'
          withCredentials([usernamePassword(credentialsId: 'nexus-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            sh """
              # Tag l'image pour le registre Nexus
              docker tag ${IMAGE_NAME} ${DOCKER_REPO}:latest

              # Login (HTTP plain possible si Nexus en http; si TLS, adapte)
              echo ${DOCKER_PASS} | docker login ${DOCKER_REPO.split('/')[0]} --username ${DOCKER_USER} --password-stdin

              # Push
              docker push ${DOCKER_REPO}:latest

              # logout
              docker logout ${DOCKER_REPO.split('/')[0]}
            """
          }
        }
      }
    }

  } // stages

  post {
    always {
      cleanWs()
    }
    success {
      echo "Pipeline finished successfully."
    }
    failure {
      echo "Pipeline failed."
    }
  }
}
