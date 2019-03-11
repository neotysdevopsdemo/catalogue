

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
    BASICCHECKURI="health"
    TAGURI="tag"
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
           sh "docker build -t ${TAG}-db:${COMMIT} $WORKSPACE/build/docker/catalogue-db/"
           sh "docker login --username=${USER} --password=${TOKEN}"
           sh "docker push ${TAG_DEV}"
           sh "docker push ${TAG}-db:${COMMIT}"

          }

      }
    }
    stage('Deploy to dev ') {


      steps {
          sh "sed -i 's,TAG_TO_REPLACE,${TAG_DEV},'  $WORKSPACE/docker-compose.yml"
          sh "sed -i 's,TAGDB_TO_REPLACE,${TAG}-db:${COMMIT},'  $WORKSPACE/docker-compose.yml"
          sh 'docker-compose -f $WORKSPACE/docker-compose.yml up -d'

      }
    }

      stage('Start NeoLoad infrastructure') {

          steps {
              sh 'docker-compose -f $WORKSPACE/infrastructure/infrastructure/neoload/lg/docker-compose.yml up -d'

          }

      }
      stage('Join Load Generators to Application') {

          steps {
              sh 'docker network connect catalogue_master_default docker-lg1'
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

    stage('Run health check in dev') {
       agent {
            dockerfile {
                args '--user root -v /tmp:/tmp --network=catalogue_master_default'
                dir 'infrastructure/infrastructure/neoload/controller/'
            }
       }
      steps {
        echo "Waiting for the service to start..."


        sleep 250

         sh "sed -i 's/CHECK_TO_REPLACE/${BASICCHECKURI}/'  $WORKSPACE/test/neoload/catalogue_neoload.yaml"
         sh "sed -i 's/TAGURL_TO_REPLACE/${TAGURI}/'  $WORKSPACE/test/neoload/catalogue_neoload.yaml"
         sh "sed -i 's/HOST_TO_REPLACE/${env.APP_NAME}/'  $WORKSPACE/test/neoload/catalogue_neoload.yaml"
         sh "sed -i 's/PORT_TO_REPLACE/80/' $WORKSPACE/test/neoload/catalogue_neoload.yaml"
         sh "sed -i 's/DTID_TO_REPLACE/${DYNATRACEID}/' $WORKSPACE/test/neoload/catalogue_neoload.yaml"
         sh "sed -i 's/APIKEY_TO_REPLACE/${DYNATRACEAPIKEY}/'  $WORKSPACE/test/neoload/catalogue_neoload.yaml"
         sh "sed -i 's,JSONFILE_TO_REPLACE,$WORKSPACE/monspec/catalogue_anomalieDection.json,'  $WORKSPACE/test/neoload/catalogue_neoload.yaml"
         sh "sed -i 's/TAGS_TO_REPLACE/${NL_DT_TAG}/' $WORKSPACE/test/neoload/catalogue_neoload.yaml"
         sh "sed -i 's,OUTPUTFILE_TO_REPLACE,$WORKSPACE/infrastructure/sanitycheck.json,'  $WORKSPACE/test/neoload/catalogue_neoload.yaml"
        script {

            neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                    project: "$WORKSPACE/test/neoload/load_template/load_template.nlp",
                    testName: 'HealthCheck_catalogue_${VERSION}_${BUILD_NUMBER}',
                    testDescription: 'HealthCheck_catalogue_${VERSION}_${BUILD_NUMBER}',
                    commandLineOption: "-project $WORKSPACE/test/neoload/catalogue_neoload.yaml -nlweb -L BasicCheck=$WORKSPACE/infrastructure/infrastructure/neoload/lg/remote.txt -L Population_Dynatrace_Integration=$WORKSPACE/infrastructure/infrastructure/neoload/lg/local.txt -nlwebToken $NLAPIKEY -variables host=${env.APP_NAME},port=80",
                    scenario: 'BasicCheck', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                    trendGraphs: [
                            [name: 'Limit test Catalogue API Response time', curve: ['CatalogueList>Actions>Get Catalogue List'], statistic: 'average'],
                            'ErrorRate'
                    ]

        }

      }
    }
    stage('Sanity Check') {
        agent {
            dockerfile {
                args '--user root -v /tmp:/tmp --network=catalogue_master_default'
                dir 'infrastructure/infrastructure/neoload/controller/'
            }
        }
              steps {
                  script {
                      neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                              project: "$WORKSPACE/test/neoload/load_template/load_template.nlp",
                              testName: 'DynatraceSanityCheck_catalogue_${VERSION}_${BUILD_NUMBER}',
                              testDescription: 'DynatraceSanityCheck_catalogue_${VERSION}_${BUILD_NUMBER}',
                              commandLineOption: "-project $WORKSPACE/test/neoload/catalogue_neoload.yaml -nlweb -L  Population_Dynatrace_SanityCheck=$WORKSPACE/infrastructure/infrastructure/neoload/lg/local.txt -variables host=${env.APP_NAME},port=80 -nlwebToken $NLAPIKEY ",
                              scenario: 'DYNATRACE_SANITYCHECK', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                              trendGraphs: [
                                      [name: 'Limit test Catalogue API Response time', curve: ['CatalogueList>Actions>Get Catalogue List'], statistic: 'average'],
                                      'ErrorRate'
                              ]
                  }



                  echo "push $WORKSPACE/infrastructure/sanitycheck.json"
                  //---add the push of the sanity check---
                  withCredentials([usernamePassword(credentialsId: 'git-credentials', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                      sh "git config --global user.email ${GIT_USERNAME}"
                      sh "git config remote.origin.url https://github.com/${env.GITHUB_ORGANIZATION}/catalogue"
                      sh "git config --add remote.origin.fetch +refs/heads/*:refs/remotes/origin/*"
                      sh "git config remote.origin.url https://github.com/${env.GITHUB_ORGANIZATION}/catalogue"
                     // sh "git add $WORKSPACE/infrastructure/sanitycheck.json"
                      //sh "git commit -m 'Update Sanity_Check_${BUILD_NUMBER} ${env.APP_NAME} '"
                      //  sh "git pull -r origin master"
                      //#TODO handle this exeption
                     // sh "git push origin HEAD:master"

                  }


              }
        }
    stage('Run functional check in dev') {
        agent {
            dockerfile {
                args '--user root -v /tmp:/tmp --network=catalogue_master_default'
                dir 'infrastructure/infrastructure/neoload/controller/'
            }
        }
      steps {
          script {
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
          }

      }
    }
    stage('Mark artifact for staging namespace') {

        steps {

            withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
                sh "docker login --username=${USER} --password=${TOKEN}"

                sh "docker tag $TAG_DEV} ${TAG_STAGING}"
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


        }

      }
}
