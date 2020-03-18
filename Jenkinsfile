

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
    DYNATRACEID="${env.DT_ACCOUNTID}.live.dynatrace.com"
    DYNATRACEAPIKEY="${env.DT_API_TOKEN}"
    NLAPIKEY="${env.NL_WEB_API_KEY}"
    NL_DT_TAG="app:${env.APP_NAME},environment:dev"
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


    /*stage('DT Deploy Event') {
        when {
            expression {
            return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
            }
        }
        steps {
          container("curl") {
            // send custom deployment event to Dynatrace
            sh "curl -X POST \"$DT_TENANT_URL/api/v1/events?Api-Token=$DT_API_TOKEN\" -H \"accept: application/json\" -H \"Content-Type: application/json\" -d \"{ \\\"eventType\\\": \\\"CUSTOM_DEPLOYMENT\\\", \\\"attachRules\\\": { \\\"tagRule\\\" : [{ \\\"meTypes\\\" : [\\\"SERVICE\\\"], \\\"tags\\\" : [ { \\\"context\\\" : \\\"CONTEXTLESS\\\", \\\"key\\\" : \\\"app\\\", \\\"value\\\" : \\\"${env.APP_NAME}\\\" }, { \\\"context\\\" : \\\"CONTEXTLESS\\\", \\\"key\\\" : \\\"environment\\\", \\\"value\\\" : \\\"dev\\\" } ] }] }, \\\"deploymentName\\\":\\\"${env.JOB_NAME}\\\", \\\"deploymentVersion\\\":\\\"${_VERSION}\\\", \\\"deploymentProject\\\":\\\"\\\", \\\"ciBackLink\\\":\\\"${env.BUILD_URL}\\\", \\\"source\\\":\\\"Jenkins\\\", \\\"customProperties\\\": { \\\"Jenkins Build Number\\\": \\\"${env.BUILD_ID}\\\",  \\\"Git commit\\\": \\\"${env.GIT_COMMIT}\\\" } }\" "
          }
        }
    }*/


    stage('Run functional check in dev') {

      steps {

         sleep 90
         sh "cp ${NEOLOAD_ANOMALIEDETECTIONFILE} $WORKSPACE/test/neoload/load_template/custom-resources/"
         sh "sed -i 's/CHECK_TO_REPLACE/${BASICCHECKURI}/'  $WORKSPACE/test/neoload/catalogue_neoload.yaml"
         sh "sed -i 's/TAGURL_TO_REPLACE/${TAGURI}/'  $WORKSPACE/test/neoload/catalogue_neoload.yaml"
         sh "sed -i 's/HOST_TO_REPLACE/${env.APP_NAME}/'  $WORKSPACE/test/neoload/catalogue_neoload.yaml"
         sh "sed -i 's/PORT_TO_REPLACE/80/' $WORKSPACE/test/neoload/catalogue_neoload.yaml"
         sh "sed -i 's/DTID_TO_REPLACE/${DYNATRACEID}/' $WORKSPACE/test/neoload/catalogue_neoload.yaml"
         sh "sed -i 's/APIKEY_TO_REPLACE/${DYNATRACEAPIKEY}/'  $WORKSPACE/test/neoload/catalogue_neoload.yaml"
         sh "sed -i 's,JSONFILE_TO_REPLACE,catalogue_anomalieDection.json,'  $WORKSPACE/test/neoload/catalogue_neoload.yaml"
         sh "sed -i 's/TAGS_TO_REPLACE/${NL_DT_TAG}/' $WORKSPACE/test/neoload/catalogue_neoload.yaml"
         sh "sed -i 's,OUTPUTFILE_TO_REPLACE,$WORKSPACE/infrastructure/sanitycheck.json,'  $WORKSPACE/test/neoload/catalogue_neoload.yaml"

         sh "mkdir $WORKSPACE/test/neoload/neoload_project"
         sh "cp $WORKSPACE/test/neoload/catalogue_neoload.yaml $WORKSPACE/test/neoload/load_template/"
         sh "cd $WORKSPACE/test/neoload/load_template/ ; zip -r $WORKSPACE/test/neoload/neoload_project/neoloadproject.zip ./*"

         sh "docker run --rm \
                                             -v $WORKSPACE/test/neoload/neoload_project/:/neoload-project \
                                             -e NEOLOADWEB_TOKEN=$NLAPIKEY \
                                             -e TEST_RESULT_NAME=FuncCheck_catalogue__${VERSION}_${BUILD_NUMBER} \
                                             -e SCENARIO_NAME=CatalogueLoad \
                                             -e CONTROLLER_ZONE_ID=defaultzone \
                                             -e LG_ZONE_IDS=defaultzone:1 \
                                             -e AS_CODE_FILES=catalogue_neoload.yaml \
                                             --network ${APP_NAME} --user root \
                                              neotys/neoload-web-test-launcher:latest"
          /*script {
              neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                      project: "$WORKSPACE/test/neoload/load_template/load_template.nlp",
                      testName: 'FuncCheck_catalogue_${VERSION}_${BUILD_NUMBER}',
                      testDescription: 'FuncCheck_catalogue_${VERSION}_${BUILD_NUMBER}',
                      commandLineOption: "-project  $WORKSPACE/test/neoload/catalogue_neoload.yaml -nlweb -L CatalogueLoad=$WORKSPACE/infrastructure/infrastructure/neoload/lg/remote.txt -L Population_Dynatrace_Integration=$WORKSPACE/infrastructure/infrastructure/neoload/lg/local.txt -nlwebToken $NLAPIKEY -variables host=catalogue,port=80",
                      scenario: 'CatalogueLoad', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                      trendGraphs: [
                              [name: 'Limit test Catalogue API Response time', curve: ['CatalogueList>Actions>Get Catalogue List'], statistic: 'average'],
                              'ErrorRate'
                      ]
          }*/

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
