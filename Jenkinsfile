pipeline {

  agent {
    kubernetes {
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    job: ${env.JOB_NAME}
    job_id: ${env.BUILD_NUMBER}
spec:
  nodeSelector:
    role: worker
  containers:
  - name: deployer
    image: quay.io/uktrade/deployer
    imagePullPolicy: Always
    command:
    - cat
    tty: true
"""
    }
  }

  options {
    timestamps()
    ansiColor('xterm')
    buildDiscarder(logRotator(daysToKeepStr: '180'))
  }

  parameters {
    string(defaultValue: 'eu-west-2', description:'Please enter region: ', name: 'cf_region')
  }

  stages {

    stage('Init') {
      steps {
        script {
          timestamps {
            validateDeclarativePipeline("${env.WORKSPACE}/Jenkinsfile")
          }
        }
      }
    }

    stage('Task') {
      steps {
        container('deployer') {
          script {
            timestamps {
              withCredentials([string(credentialsId: env.GDS_PAAS_CONFIG, variable: 'paas_config_raw')]) {
                paas_config = readJSON text: paas_config_raw
              }
              if (!params.cf_region) {
                cf_region = paas_config.default
              }
              paas_region = paas_config.regions."${cf_region}"
              echo "\u001B[32mINFO: Setting PaaS region to ${paas_region.name}.\u001B[m"

              withCredentials([usernamePassword(credentialsId: paas_region.credential, passwordVariable: 'gds_pass', usernameVariable: 'gds_user')]) {
                sh """
                  cf api ${paas_region.api}
                  cf auth ${gds_user} ${gds_pass}
                """
              }

              conduit_ids_json = sh(script: "cf curl '/v3/apps?per_page=5000' | jq '[.resources[] | select(.name|test(\"__conduit_[0-9a-zA-Z]+__\")) | select(now - (.updated_at|fromdateiso8601) > 43200) | .guid]'", returnStdout: true).trim()
              conduit_ids = readJSON text: conduit_ids_json
              conduit_ids.each {
                sh "cf curl '/v3/apps/${it}' -X DELETE || true"
              }
            }
          }
        }
      }
    }

  }
}
