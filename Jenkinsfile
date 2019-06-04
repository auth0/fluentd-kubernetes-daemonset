pipeline {
  agent {
    label 'crew-platform' // Each crew has an agent/label available (crew-2, crew-apollo, crew-appliance, crew-auth, crew-brokkr, crew-brucke, crew-guardian, crew-services, crew-skynet, crew-mktg, cs-infra)
  }

  environment { // This block defines environment variables that will be available throughout the rest of the pipeline
    SERVICE_NAME = 'fluentd-kubernetes-daemonset'
    DOCKERFILE = 'v1.4/debian-datadog'
  }

  options {
    timeout(time: 10, unit: 'MINUTES') // Global timeout for the job. Recommended to make the job fail if it's taking too long
  }

  parameters { // Job parameters that need to be supplied when the job is run. If they have a default value they won't be required
    string(name: 'SlackTarget', defaultValue: '#platform-deployments', description: 'Target Slack Channel for notifications')
  }

  stages {
    stage('SharedLibs') { // Required. Stage to load the Auth0 shared library for Jenkinsfile
      steps {
        library identifier: 'auth0-jenkins-pipelines-library@master', retriever: modernSCM(
          [$class: 'GitSCMSource',
          remote: 'git@github.com:auth0/auth0-jenkins-pipelines-library.git',
          credentialsId: 'auth0extensions-ssh-key'])
      }
    }

    stage('Build Docker Image') {
      steps {
        script {
          DOCKER_REGISTRY = getDockerRegistry()
          DOCKER_REPO = "${DOCKER_REGISTRY}/${env.SERVICE_NAME}"
          DOCKER_TAG = getDockerTag()
          sh "make image tags no-cache=yes IMAGE_NAME=${DOCKER_REPO} TAGS=${DOCKER_TAG} DOCKERFILE=${DOCKERFILE}"
        }
      }
    }

    stage('Push Docker Image') {
      steps {
        withDockerRegistry(getArtifactoryRegistry()) {
          sh "make push IMAGE_NAME=${DOCKER_REPO} TAGS=${DOCKER_TAG}"
        }
      }
    }

    stage('Update "latest" Tag') {
      when {
        branch 'auth0-integrations'
      }
      steps {
        script {
          withDockerRegistry(getArtifactoryRegistry()) {
            docker.image("${DOCKER_REPO}:${DOCKER_TAG}").push('latest')
            sh "make tags push IMAGE_NAME=${DOCKER_REPO}"
          }
        }
      }
    }
  }

  post {
    always { // Steps that need to run regardless of the job status, such as test results publishing, Slack notifications or dependencies cleanup
      script {
        String additionalMessage = ''
        String buildResult = currentBuild.result ?: 'SUCCESS'
        if (buildResult == 'SUCCESS') {
          additionalMessage += "\n```pushed: ${DOCKER_REPO}:${DOCKER_TAG}```"
        }
        notifySlack(params.SlackTarget, additionalMessage)
      }
    }
    cleanup {
      // Recommended to clean the workspace after every run
      deleteDir()
      dockerRemoveImage("${DOCKER_REPO}:${DOCKER_TAG}")
    }
  }
}
