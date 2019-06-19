pipeline {

  agent {
    node {
      label env.CI_SLAVE
    }
  }

  options {
    timestamps()
    ansiColor('xterm')
  }

  parameters {
    string(defaultValue: 'eu-west-2', description:'Please enter region: ', name: 'cf_region')
  }

  stages {

    stage('Init') {
      steps {
        script {
          validateDeclarativePipeline("${env.WORKSPACE}/Jenkinsfile")
          deployer = docker.image("quay.io/uktrade/deployer:${env.GIT_BRANCH.split("/")[1]}")
          docker_args = "--network host"
          deployer.pull()
        }
      }
    }

    stage('Task') {
      steps {
        script {
          deployer.inside(docker_args) {
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

            conduit_ids_json = sh(script: "cf curl '/v3/apps?per_page=5000' | jq '[.resources[] | select(.name|test(\"__conduit_[0-9]+__\")) | select(now - (.updated_at|fromdateiso8601) > 43200) | .guid]'", returnStdout: true).trim()
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
