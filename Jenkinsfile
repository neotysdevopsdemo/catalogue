



pipeline {
  agent  { label 'master' }
      tools {
          go 'go'
          jdk 'jdk8'
      }
  environment {
    VERSION="0.1"
    APP_NAME = "catalogue"
    TAG = "neotysdevopsdemo/${APP_NAME}"
    TAG_DEV = "${TAG}:DEV-${VERSION}"
    ARTEFACT_ID = "sockshop-" + "${env.APP_NAME}"
    TAG_STAGING = "${TAG}-stagging:${VERSION}"
    DYNATRACEID="https://${env.DT_ACCOUNTID}.live.dynatrace.com/"
    DYNATRACEAPIKEY="${env.DT_API_TOKEN}"
    NLAPIKEY="${env.NL_WEB_API_KEY}"
    NL_DT_TAG_APP="app:${env.APP_NAME}"
    NL_DT_TAG_ENV="environment:dev"
    OUTPUTSANITYCHECK="$WORKSPACE/infrastructure/sanitycheck.json"
    NEOLOAD_ASCODEFILE="$WORKSPACE/test/neoload/catalogue_neoload.yaml"
    NEOLOAD_ANOMALIEDETECTIONFILE="$WORKSPACE/monspec/catalogue_anomalieDection.json"
    DOCKER_COMPOSE_TEMPLATE="$WORKSPACE/infrastructure/infrastructure/neoload/docker-compose.template"
    DOCKER_COMPOSE_LG_FILE = "$WORKSPACE/infrastructure/infrastructure/neoload/docker-compose-neoload.yml"
    BASICCHECKURI="health"
    TAGURI="tags"
    GROUP = "neotysdevopsdemo"
    COMMIT = "DEV-${VERSION}"
  }
  stages {
      stage('Checkout') {
          agent { label 'master' }
          steps {
              git  url:"https://github.com/${GROUP}/${APP_NAME}.git",
                      branch :'master'
          }
      }
    stage('Go build') {
      steps {

       /* script {
          _VERSION = readFile('version').trim()
          _TAG = "${env.DOCKER_REGISTRY_URL}:5000/sockshop-registry/${env.ARTEFACT_ID}"
          _TAG_DEV = "${_TAG}:${_VERSION}-${env.BUILD_NUMBER}"
          _TAG_STAGING = "${_TAG}:${_VERSION}"
        }*/

          sh '''

            export CODE_DIR=$PWD

            export GOPATH=$PWD

           mkdir -p src/github.com/neotysdevopsdemo/catalogue/
           go get -v github.com/Masterminds/glide
           cp -R ./api src/github.com/neotysdevopsdemo/catalogue/
           cp -R ./main.go src/github.com/neotysdevopsdemo/catalogue/
           cp -R ./glide.* src/github.com/neotysdevopsdemo/catalogue/
           cd src/github.com/neotysdevopsdemo/catalogue



           '''
     

      }
    }
    stage('Docker build') {

      steps {
          withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
           sh "docker pull ${GROUP}/${APP_NAME}:DEV-0.1"
           sh "docker tag ${GROUP}/${APP_NAME}:DEV-0.1 ${TAG_DEV}"
           sh "docker build -t ${TAG}-db:${COMMIT} $WORKSPACE/docker/catalogue-db/"
           sh "docker login --username=${USER} --password=${TOKEN}"
           sh "docker push ${TAG_DEV}"
           sh "docker push ${TAG}-db:${COMMIT}"

          }

      }
    }

     stage('create docker netwrok') {

                                  steps {
                                       sh "docker network create ${APP_NAME} || true"

                                  }
                   }
    stage('Deploy to dev ') {


      steps {
          sh "sed -i 's,TAG_TO_REPLACE,${TAG_DEV},'  $WORKSPACE/docker-compose.yml"
          sh "sed -i 's,TAGDB_TO_REPLACE,${TAG}-db:${COMMIT},'  $WORKSPACE/docker-compose.yml"
          sh "sed -i 's,TO_REPLACE,${APP_NAME},' $WORKSPACE/docker-compose.yml"
          sh 'docker-compose -f $WORKSPACE/docker-compose.yml up -d'

      }
    }

     stage('Start NeoLoad infrastructure') {

                                steps {
                                           sh "cp -f ${DOCKER_COMPOSE_TEMPLATE} ${DOCKER_COMPOSE_LG_FILE}"
                                           sh "sed -i 's,TO_REPLACE,${APP_NAME},'  ${DOCKER_COMPOSE_LG_FILE}"
                                           sh "sed -i 's,TOKEN_TOBE_REPLACE,$NLAPIKEY,'  ${DOCKER_COMPOSE_LG_FILE}"
                                           sh 'docker-compose -f ${DOCKER_COMPOSE_LG_FILE} up -d'
                                           sleep 15

                                       }

                           }


      stage('NeoLoad Test')
            {
             agent {
             docker {
                 image 'python:3-alpine'
                 reuseNode true
              }

                }
            stages {
                 stage('Get NeoLoad CLI') {
                              steps {
                                withEnv(["HOME=${env.WORKSPACE}"]) {

                                 sh '''
                                      export PATH=~/.local/bin:$PATH
                                      pip3 install neoload
                                      neoload --version
                                  '''

                                }
                              }
                }


                stage('Run functional check in dev') {

                  steps {
                    withEnv(["HOME=${env.WORKSPACE}"]) {
                     sleep 90
                     sh "sed -i 's/CHECK_TO_REPLACE/${BASICCHECKURI}/'  $WORKSPACE/test/neoload/catalogue_neoload.yaml"
                     sh "sed -i 's/TAGURL_TO_REPLACE/${TAGURI}/'  $WORKSPACE/test/neoload/catalogue_neoload.yaml"
                     sh "sed -i 's/HOST_TO_REPLACE/${env.APP_NAME}/'  $WORKSPACE/test/neoload/catalogue_neoload.yaml"
                     sh "sed -i 's/PORT_TO_REPLACE/80/' $WORKSPACE/test/neoload/catalogue_neoload.yaml"
                     sh "sed -i 's,DTID_TO_REPLACE,${DYNATRACEID},' $WORKSPACE/test/neoload/catalogue_neoload.yaml"
                     sh "sed -i 's/APIKEY_TO_REPLACE/${DYNATRACEAPIKEY}/'  $WORKSPACE/test/neoload/catalogue_neoload.yaml"
                     sh "sed -i 's/TAGS_TO_APP/${NL_DT_TAG_APP}/' $WORKSPACE/test/neoload/catalogue_neoload.yaml"
                     sh "sed -i 's/TAGS_ENV/${NL_DT_TAG_ENV}/' $WORKSPACE/test/neoload/catalogue_neoload.yaml"



                    sh """
                           export PATH=~/.local/bin:$PATH

                           neoload \
                           login --workspace "Default Workspace" $NLAPIKEY \
                           test-settings  --zone defaultzone --scenario CatalogueLoad use CatalogueDynatrace \
                           project --path $WORKSPACE/test/neoload upload
                      """

                    }
                  }
                }
                 stage('Run Test') {
                                  steps {
                                    withEnv(["HOME=${env.WORKSPACE}"]) {
                                      sh """
                                           export PATH=~/.local/bin:$PATH
                                           neoload run \
                                          --return-0 \
                                            --as-code catalogue_neoload.yaml \
                                            CatalogueDynatrace
                                         """
                                    }
                                  }
                            }

            }
    }
    stage('Mark artifact for staging namespace') {

        steps {

            withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
                sh "docker login --username=${USER} --password=${TOKEN}"

                sh "docker tag ${TAG_DEV} ${TAG_STAGING}"
                sh "docker push ${TAG_STAGING}"
                sh "docker tag ${TAG}-db:${COMMIT} ${TAG}-db-stagging:${VERSION}"
                sh "docker push ${TAG}-db-stagging:${VERSION}"
            }

        }
    }

  }
  post {
        always {
                sh 'docker-compose -f $WORKSPACE/infrastructure/infrastructure/neoload/lg/docker-compose.yml down'
                sh 'docker-compose -f $WORKSPACE/docker-compose.yml down'
                cleanWs()
                sh 'docker volume prune'
        }

      }
}
