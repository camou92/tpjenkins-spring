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
  }

  stages {

    stage("Clean up"){
      steps {
        deleteDir()
      }
    }

    stage("Clone repo"){
      steps {
        sh "git clone ${GIT_URL}"
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

    stage("Automatic Maven Release (prepare & perform)"){
      steps {
        dir("tpjenkins-spring") {
          withCredentials([
            usernamePassword(credentialsId: 'github', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS'),
            usernamePassword(credentialsId: 'nexus-cred', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')
          ]) {
            sh '''
              # Configure Git author for release plugin
              git config user.email "cmohamed992@gmail.com"
              git config user.name "camou92"

              # Set remote URL with embedded credentials for push
              git remote set-url origin https://${GIT_USER}:${GIT_PASS}@${GIT_URL.replaceFirst('https://','')}

              # Prepare and perform release (skip tests during release)
              mvn -B -s settings.xml release:prepare -Darguments="-DskipTests" -Dresume=false
              mvn -B -s settings.xml release:perform -Darguments="-DskipTests"
            '''
          }
        }
      }
    }

    stage("Capture Release Version"){
      steps {
        dir("tpjenkins-spring") {
          sh '''
            if [ -f release.properties ]; then
              grep '^releaseVersion=' release.properties | sed 's/releaseVersion=//' > .release_version
            else
              git fetch --tags
              git describe --tags --abbrev=0 > .release_version || echo "unknown" > .release_version
            fi
            cat .release_version
          '''
          script {
            def ver = readFile('tpjenkins-spring/.release_version').trim()
            if (ver == '' || ver == 'unknown') {
              error "Cannot determine release version."
            }
            env.RELEASE_VERSION = ver
          }
        }
      }
    }

    stage("Build Docker Image"){
      steps {
        dir("tpjenkins-spring") {
          sh '''
            docker build -t ${IMAGE_NAME}:${RELEASE_VERSION} .
            docker tag ${IMAGE_NAME}:${RELEASE_VERSION} ${DOCKER_REPO}:latest
            docker tag ${IMAGE_NAME}:${RELEASE_VERSION} ${DOCKER_REPO}:${RELEASE_VERSION}
          '''
        }
      }
    }

    stage("Push Docker Image to Nexus"){
      steps {
        dir("tpjenkins-spring") {
          withCredentials([usernamePassword(credentialsId: 'nexus-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            sh '''
              echo $DOCKER_PASS | docker login ${DOCKER_REPO.split('/')[0]} --username $DOCKER_USER --password-stdin
              docker push ${DOCKER_REPO}:${RELEASE_VERSION}
              docker push ${DOCKER_REPO}:latest
              docker logout ${DOCKER_REPO.split('/')[0]}
            '''
          }
        }
      }
    }

    stage("Deploy & Run Application with Docker Compose"){
      steps {
        dir("tpjenkins-spring") {
          withCredentials([usernamePassword(credentialsId: 'nexus-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            sh '''
              echo $DOCKER_PASS | docker login ${DOCKER_REPO.split('/')[0]} --username $DOCKER_USER --password-stdin
              docker compose down --remove-orphans || true
              docker pull ${DOCKER_REPO}:${RELEASE_VERSION}
              docker pull ${DOCKER_REPO}:latest
              # Optionnel: injecter la variable RELEASE_VERSION dans docker-compose via --env-file ou override
              docker compose up -d
              docker logout ${DOCKER_REPO.split('/')[0]}
            '''
          }
        }
      }
    }

  } // end stages

  post {
    always {
      cleanWs()
    }
    success {
      echo "🚀 Pipeline SUCCESS — Release ${env.RELEASE_VERSION} deployed with Docker."
    }
    failure {
      echo "❌ Pipeline FAILED — Check logs."
    }
  }

}
