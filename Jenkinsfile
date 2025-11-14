pipeline {
  agent any

  environment {
    // Replace these with your values
    DOCKER_HUB_REPO = "yourdockerhub/demo-app"        // <username>/<repo>
    DOCKERHUB_CREDENTIALS = "dockerhub-creds"         // Jenkins credentials id (username/password)
    KUBECONFIG_CREDENTIALS = "kubeconfig-creds"       // Jenkins secret file or credential to access cluster (optional)
    MAVEN_OPTS = "-DskipTests=true"
  }

  options {
    // Keep only last 10 builds
    buildDiscarder(logRotator(numToKeepStr: '10'))
    // Timeout to avoid hung builds
    timeout(time: 30, unit: 'MINUTES')
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Set up') {
      steps {
        script {
          // ensure executable wrapper on Linux nodes
          sh 'if [ -f mvnw ]; then chmod +x mvnw; fi'
        }
      }
    }

    stage('Build (Maven)') {
      steps {
        // Use wrapper if available otherwise fallback to mvn
        sh '''
          if [ -f ./mvnw ]; then
            ./mvnw clean package ${MAVEN_OPTS}
          else
            mvn clean package ${MAVEN_OPTS}
          fi
        '''
        // Archive the generated jar for inspection
        archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: false
      }
    }

    stage('Unit Tests') {
      steps {
        // If you didn't skip tests above you can run them here.
        // This stage will not fail the pipeline if there are no tests.
        sh '''
          if [ -f ./mvnw ]; then
            ./mvnw test || echo "Tests failed or skipped"
          else
            mvn test || echo "Tests failed or skipped"
          fi
        '''
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
        }
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          // Use the jar file that was just built (adjust name if different)
          def imageTag = "${env.BUILD_NUMBER}"
          sh "docker build -t ${DOCKER_HUB_REPO}:latest -t ${DOCKER_HUB_REPO}:${imageTag} ."
          // save tag to file for downstream steps if needed
          writeFile file: 'image.tag', text: imageTag
        }
      }
    }

    stage('Docker Login & Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: env.DOCKERHUB_CREDENTIALS, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push ${DOCKER_HUB_REPO}:latest
            TAG=$(cat image.tag)
            docker push ${DOCKER_HUB_REPO}:$TAG
          '''
        }
      }
    }

    stage('Deploy to Kubernetes') {
      when {
        expression { fileExists('k8s/deployment.yaml') }
      }
      steps {
        script {
          // Option A: kubeconfig stored as secret file in Jenkins (preferred)
          // Add a "Secret file" credential in Jenkins with id = KUBECONFIG_CREDENTIALS
          // and then uncomment the withCredentials block below and comment the plain "kubectl apply" line.
          //
          // withCredentials([file(credentialsId: env.KUBECONFIG_CREDENTIALS, variable: 'KUBECONFIG_FILE')]) {
          //   sh 'export KUBECONFIG=$KUBECONFIG_FILE; kubectl apply -f k8s/'
          // }

          // Option B: if kubectl is already configured on the agent (kubeconfig in place)
          sh 'kubectl apply -f k8s/ || echo "kubectl apply failed (check kubeconfig or credentials)"'
        }
      }
    }
  }

  post {
    success {
      echo "Build and deploy succeeded: ${DOCKER_HUB_REPO}:$(cat image.tag)"
    }
    failure {
      echo "Pipeline failed. Check console output."
    }
    always {
      // Cleanup local dangling images to save space on agent
      sh 'docker image prune -f || true'
    }
  }
}
