if (scm.branches[0].name == 'master') {
  properties(
		[
			pipelineTriggers([cron('H 7 1 * *')])
		]
	)
}
pipeline {
  options {
    timeout(time: 1, unit: 'HOURS')
  }
  agent {
    node {
      label 'jenkins_master'
    }
  }
  stages {
    stage('Pre-Build') {
      steps {
        script {
          slackResponse = slack.threaded(channel: "#pdt-pipeline", message: "*Pipeline Started*\n${env.SLACK_PIPELINE_INFO}")
          cleanedJobName = "${env.JOB_NAME}".toLowerCase().replaceAll("%2f", "/")
          if ("${env.BRANCH_NAME}" == "master") {
            dockerImageTag = 'latest'
          }
          if ("${env.BRANCH_NAME}" != "master") {
            dockerImageTag = 'beta'
          }
        }
      }
    }
    stage('Validate Templates') {
      steps {
        script {
          credentials.pipeline(region: 'us-west-2') {
            cfnValidate(file: 'templates/spark-glue-base-ecr-repo.template')
          }
        }
      }
    }
    stage('Deploy Spark-Glue docker to phtech-pipeline Account West') {
      steps {
        script {
          credentials.pipeline(region: 'us-west-2') {
            westECSRepoStack = cfnUpdate(
              stack: 'Spark-Glue-ECS-Base-Repo',
              create: true,
              timeoutInMinutes: 10,
              pollInterval:10000,
              tags: [
                "Role=DockerBaseRepo",
                "JenkinsJobName=${cleanedJobName}"
              ],
              file: 'templates/spark-glue-base-ecr-repo.template',
              params: ["RootAccountARNList=${AWS_ECR_ALLOWED_ACCOUNTS}"]
            )
          }
        }
      }
    }
    stage('Deploy Linux Base Repo to phtech-pipeline Account East') {
      steps {
        script {
          credentials.pipeline(region: 'us-east-1') {
            eastECSRepoStack = cfnUpdate(stack: 'Spark-Glue-ECS-Base-Repo',
            create: true,
            timeoutInMinutes: 10,
            pollInterval:10000,
            tags: [
              "Role=DockerBaseRepo",
              "JenkinsJobName=${cleanedJobName}"
            ],
            file: 'templates/spark-glue-base-ecr-repo.template',
            params: ["RootAccountARNList=${AWS_ECR_ALLOWED_ACCOUNTS}"]
            )
          }
        }
      }
    }
    stage('Build Docker Base Image'){
      agent{
        node {
          label 'linux_docker_build'
        }
      }
      steps{
        script {
          credentials.pipeline(region: 'us-west-2') {
            def login = ecrLogin()
            sh "${login}"
            sh "docker build --build-arg JENKINS_MASTER_HOST=${env.JENKINS_URL} -t spark-glue-base:${env.BUILD_NUMBER} ."
            sh "docker tag spark-glue-base:${env.BUILD_NUMBER} ${westECSRepoStack['RepoURI']}:${dockerImageTag}"
            sh "docker push ${westECSRepoStack['RepoURI']}:${dockerImageTag}"
          }
          if ("${env.BRANCH_NAME}" == "master") {
            credentials.pipeline(region: 'us-east-1') {
              def login = ecrLogin()
              sh "${login}"
              sh "docker tag spark-glue-base:${env.BUILD_NUMBER} ${eastECSRepoStack['RepoURI']}:${dockerImageTag}"
              sh "docker push ${eastECSRepoStack['RepoURI']}:${dockerImageTag}"
            }
          }
        }
      }
    }
  }
  environment {
    AWS_INFORMATICS_ACCOUNT_ID = credentials('php-informatics-prod_AccountID')
    AWS_PIPELINE_ACCOUNT_ID = credentials('phtech-pipeline_AccountID')
    AWS_ECR_ALLOWED_ACCOUNTS = "arn:aws:iam::${AWS_PIPELINE_ACCOUNT_ID}:root,arn:aws:iam::${AWS_INFORMATICS_ACCOUNT_ID}:root"
    SLACK_PIPELINE_INFO = ">>>Job Name: *<${env.JOB_DISPLAY_URL}|${env.JOB_NAME.toLowerCase().replaceAll('%2f','/')}>*\nBuild Number: *${env.BUILD_NUMBER}* (<${RUN_DISPLAY_URL}|See Details>)\n<${RUN_CHANGES_DISPLAY_URL}|Associated Changes>"
  }
  post {
    success {
      script {
        slack.threaded(channel: "${slackResponse.threadId}", message: "*Pipeline status: ${currentBuild.currentResult}*\n${env.SLACK_PIPELINE_INFO}")
      }
    }
    unsuccessful {
      script {
        slack.threaded(channel: "${slackResponse.threadId}", message: "*Pipeline status: ${currentBuild.currentResult}*\n${env.SLACK_PIPELINE_INFO}")
      }
    }
  }
}