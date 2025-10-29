pipeline {
  agent any

  environment {
    // 🔧 Jenkins credentials ID for AWS ECR (Access Key + Secret)
    AWS_CREDS = credentials('ecr-creds')

    // 🪣 AWS region & repo details
    AWS_REGION = "us-east-1" // Change if your ECR is in a different region
    ECR_REPO = "123456789012.dkr.ecr.us-east-1.amazonaws.com/siddharth-test" // replace with your actual ECR repo URI
  }

  options {
    timestamps()
  }

  stages {

    stage('Checkout') {
      steps {
        echo "📦 Cloning repository..."
        git branch: 'main',
            url: 'https://github.com/siddharth0203/DemoPythonJenkins.git',
            credentialsId: 'github-token'
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          echo "🐳 Building Docker image..."
          sh """
            docker build -t generic-app:${BUILD_NUMBER} .
            docker tag generic-app:${BUILD_NUMBER} ${ECR_REPO}:${BUILD_NUMBER}
          """
        }
      }
    }

    stage('Scan Docker Image (Trivy)') {
      steps {
        script {
          echo "🔍 Scanning Docker image..."
          sh '''
            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
              aquasec/trivy:latest image \
              --exit-code 0 --no-progress --format json generic-app:${BUILD_NUMBER} > scan.json || true
          '''
          def vulnCount = sh(script: "cat scan.json | jq '.Results[0].Vulnerabilities | length' 2>/dev/null || echo 0", returnStdout: true).trim()
          echo "🧮 Vulnerabilities found: ${vulnCount}"
          env.VULN_COUNT = vulnCount
        }
      }
    }

    stage('Evaluate Scan Results') {
      steps {
        script {
          def score = (env.VULN_COUNT ?: '0').toInteger()
          if (score >= 5) {
            echo "❌ Too many vulnerabilities (${score} >= 5). Aborting push."
            currentBuild.result = 'ABORTED'
            error("Build stopped due to high vulnerability count.")
          } else {
            echo "✅ Vulnerability score acceptable (${score} < 5). Proceeding to push."
          }
        }
      }
    }

    stage('Push to AWS ECR') {
      when {
        expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
      }
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${AWS_CREDS}"]]) {
          script {
            echo "🔐 Logging in to AWS ECR..."
            sh """
              aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
              docker push ${ECR_REPO}:${BUILD_NUMBER}
            """
          }
        }
      }
    }
  }

  post {
    always {
      echo "Pipeline completed with status: ${currentBuild.result ?: 'SUCCESS'}"
      archiveArtifacts artifacts: 'scan.json', allowEmptyArchive: true
      sh 'docker image rm generic-app:${BUILD_NUMBER} || true'
      sh "docker image rm ${ECR_REPO}:${BUILD_NUMBER} || true"
    }
    success {
      echo "🎉 Image pushed successfully to AWS ECR!"
    }
    aborted {
      echo "⚠️ Pipeline aborted due to vulnerabilities."
    }
    failure {
      echo "❌ Pipeline failed."
    }
  }
}
