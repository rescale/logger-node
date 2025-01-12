library 'magic-butler-catalogue'

def PROJECT_NAME = "logger-node"
def REPO = "logdna/${PROJECT_NAME}"
def TRIGGER_PATTERN = ".*@logdnabot.*"

pipeline {
  agent none

  options {
    timestamps()
    ansiColor 'xterm'
  }

  triggers {
    issueCommentTrigger(TRIGGER_PATTERN)
  }

  stages {
    stage('Validate PR Source') {
      when {
        expression { env.CHANGE_FORK }
        not {
          triggeredBy 'issueCommentCause'
        }
      }
      steps {
        error("A maintainer needs to approve this PR for CI by commenting")
      }
    }

    stage('Test Suite') {
      matrix {
        axes {
          axis {
            name 'NODE_VERSION'
            values '10', '12', '14'
          }
        }

        agent {
          docker {
            image "us.gcr.io/logdna-k8s/node:${NODE_VERSION}-ci"
          }
        }

        environment {
          NPM_CONFIG_CACHE = '.npm'
          NPM_CONFIG_USERCONFIG = '.npm/rc'
          SPAWN_WRAP_SHIM_ROOT = '.npm'
        }

        stages {
          stage('Install') {
            environment {
              GITHUB_PACKAGES_TOKEN = credentials('github-api-token')
            }

            steps {
              sh 'mkdir -p .npm'

              script {
                npm.auth token: "${GITHUB_PACKAGES_TOKEN}"
              }

              sh """
              npm install
              """
            }
          }

          stage('Test') {
            steps {
              sh 'node -v'
              sh 'npm -v'
              sh 'npm test'
            }

            post {
              always {
                sh 'cat .tap-output | ./node_modules/.bin/tap-mocha-reporter xunit > coverage/test.xml'

                junit 'coverage/test.xml'

                publishHTML target: [
                  allowMissing: false,
                  alwaysLinkToLastBuild: false,
                  keepAll: true,
                  reportDir: 'coverage/lcov-report',
                  reportFiles: 'index.html',
                  reportName: "coverage-node-v${NODE_VERSION}"
                ]
              }
            }
          }
        }
      }
    }

    stage('Test Release') {
      when {
        beforeAgent true
        not {
          branch 'master'
        }
      }

      agent {
        docker {
          image "us.gcr.io/logdna-k8s/node:12-ci"
          customWorkspace "${PROJECT_NAME}-${BUILD_NUMBER}"
        }
      }

      environment {
        NPM_CONFIG_CACHE = '.npm'
        NPM_CONFIG_USERCONFIG = '.npm/rc'
        SPAWN_WRAP_SHIM_ROOT = '.npm'
        GITHUB_PACKAGES_TOKEN = credentials('github-api-token')
      }

      steps {
        sh 'mkdir -p .npm'

        versioner(
          token: "${GITHUB_PACKAGES_TOKEN}"
        , dry: true
        , repo: REPO
        , branch: "master"
        )
      }
    }

    stage('Release') {
      when {
        beforeAgent true
        branch 'master'
      }

      agent {
        docker {
          image "us.gcr.io/logdna-k8s/node:12-ci"
          customWorkspace "${PROJECT_NAME}-${BUILD_NUMBER}"
        }
      }

      environment {
        GITHUB_PACKAGES_TOKEN = credentials('github-api-token')
        NPM_PUBLISH_TOKEN = credentials('npm-publish-token')
        NPM_CONFIG_CACHE = '.npm'
        NPM_CONFIG_USERCONFIG = '.npm/rc'
        SPAWN_WRAP_SHIM_ROOT = '.npm'
      }

      steps {
        sh 'mkdir -p .npm'

        sh "git checkout -b ${GIT_BRANCH} origin/${GIT_BRANCH}"

        versioner(
          token: "${GITHUB_PACKAGES_TOKEN}"
        , dry: false
        , repo: REPO
        , NPM_PUBLISH_TOKEN: "${NPM_PUBLISH_TOKEN}"
        , branch: "master"
        )
      }
    }
  }
}
