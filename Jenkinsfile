node {
    environment {
        APP_NAME = "SampleNodeJs"
        STAGING = "Staging"
        PRODUCTION = "Production"
        execution_id = 0
    } 
    
    stage('Checkout') {
        // Checkout our application source code
        git url: 'https://github.com/nikhilgoenkatech/JenkinsBankApp.git'
    }

    stage('Build') {
            steps {
                script {
                    def buildNumber = env.BUILD_NUMBER
                    def userName = currentBuild.rawBuild.getCause(hudson.model.Cause$UserIdCause).getUserName()
                    def logFile = "build-${buildNumber}.log"  // Define the name of the log file
                    
                    // Execute the 'dir' block and capture its output
                    def buildOutput = bat(script: """
                        cd sample-bank-app-service
                        try {
                            env.DOCKERFILE = env.DOCKERFILE
                        }
                        catch (e) {
                            echo "Received an exception!!!"
                            env.DOCKERFILE = "Dockerfile"
                        }
                        docker build -t sample-bankapp-service:${buildNumber} -f ${env.DOCKERFILE} .
                    """, returnStatus: true)
                    
                    // Prepend build number and user to each line and write to the log file
                    writeFile file: "${logFile}", text: buildOutput.split('\n').collect {
                        "[Build ${buildNumber} by ${userName}] ${it}"
                    }.join('\n')
                    
                    // Optionally, archive the log file for the Build stage
                    archiveArtifacts artifacts: "${logFile}", allowEmptyArchive: true
                }
            }
        
        // Lets build our docker image
        //dir ('sample-bank-app-service') {
        //    try {
        //        env.DOCKERFILE = env.DOCKERFILE
        //    }
        //    catch (e) {
            //catch (groovy.lang.MissingPropertyException e ) {
         //       echo "Received an exception!!!"
         //       env.DOCKERFILE = "Dockerfile"
         //   }
          //  def app = docker.build("sample-bankapp-service:${BUILD_NUMBER}", "-f ${env.DOCKERFILE} .")
        //}
    //}
    
    stage('CleanStaging') {
        // The cleanup script makes sure no previous docker staging containers run
        sh 'docker ps -f name=SampleOnlineBankStaging -q | xargs --no-run-if-empty docker container stop'
        sh 'docker container ls -a -fname=SampleOnlineBankStaging -q | xargs -r docker container rm'
    }
    
    stage('DeployStaging') {
        // Lets deploy the previously build container
        def app = docker.image("sample-bankapp-service:${BUILD_NUMBER}")
        app.run("--network mynetwork --name SampleOnlineBankStaging -p 3000:3000 " +
                "-e 'DT_CLUSTER_ID=SampleOnlineBankStaging' " + 
                "-e 'DT_TAGS=Environment=Staging Service=Sample-NodeJs-Service' " +
                "-e 'DT_CUSTOM_PROP=ENVIRONMENT=Staging JOB_NAME=${JOB_NAME} " + 
                    "BUILD_TAG=${BUILD_TAG} BUILD_NUMBER=${BUIlD_NUMBER}'")

        dir ('dynatrace-scripts') {
            // push a deployment event on the host with the tag JenkinsInstance created using automatic tagging rule
            sh './pushdeployment.sh HOST JenkinsInstance ' +
               '${BUILD_TAG} ${BUILD_NUMBER} ${JOB_NAME} ' + 
               'Staging SampleOnlineBankStaging'
                        
            // Create a on-demand synthetic monitor so as to check the UI functionlity
            sh './synthetic-monitor.sh Staging '+  '${JOB_NAME} ${BUILD_NUMBER}' + ' 3000'
            
            // Create SLOs for the staging environment
            sh "python3 create_slo.py ${DT_URL} ${DT_TOKEN} SampleOnlineBankStaging DockerService staging"
            
            // Pull the SLOs id and create a sample dashboard for the staging stage
            sh "python3 populate_slo.py ${DT_URL} ${DT_TOKEN} SampleOnlineBankStaging ${JOB_NAME} staging ${BUILD_NUMBER} DockerService"
        }
    }
    
    stage('Testing') {
        // lets push an event to dynatrace that indicates that we START a load test
        dir ('dynatrace-scripts') {
            sh './pushevent.sh SERVICE DockerService SampleOnlineBankStaging ' +
               '"STARTING Load Test" ${JOB_NAME} "Starting a Load Test as part of Staging"' + 
               ' ${JENKINS_URL} ${JOB_URL} ${BUILD_URL} ${GIT_COMMIT}'
        }
        dir('dynatrace-scripts') {
            // Trigger the on-demand synthetic monitor as part of the Testing cycle
            env.execution_id = sh(script: 'python3 trigger_syn_monitor.py ${DT_URL} ${DT_TOKEN} Staging ${BUILD_NUMBER}', returnStatus: true)

        }
        
        // lets run some test scripts
        dir ('sample-bank-app-service-tests') {
            // start load test - simulating traffic for Staging enviornment on port 3000 

            sh "rm -f stagingloadtest.log stagingloadtestcontrol.txt"
            sh "python3 smoke-test.py 3000 10 ${BUILD_NUMBER} stagingloadtest.log ${PUBLIC_IP} SampleOnlineBankStaging"
            archiveArtifacts artifacts: 'stagingloadtest.log', fingerprint: true
        }

        // lets push an event to dynatrace that indicates that we STOP a load test
        dir ('dynatrace-scripts') {
            sh './pushevent.sh SERVICE DockerService SampleOnlineBankStaging '+
               '"STOPPING Load Test" ${JOB_NAME} "Stopping a Load Test as part of the Testing stage" '+
               '${JENKINS_URL} ${JOB_URL} ${BUILD_URL} ${GIT_COMMIT}'
        }

        // lets push an event to dynatrace that indicates that we START a sanity test
        dir ('dynatrace-scripts') {
            sh './pushevent.sh SERVICE DockerService SampleOnlineBankStaging ' +
               '"STARTING Sanity-Test" ${JOB_NAME} "Starting Sanity-test of the Testing stage"' + 
               ' ${JENKINS_URL} ${JOB_URL} ${BUILD_URL} ${GIT_COMMIT}'
        }
        
        // lets run some test scripts
        dir ('sample-bank-app-service-tests') {
            // start load test - simulating traffic for Staging enviornment on port 3000 

            sh "rm -f stagingloadtest.log stagingloadtestcontrol.txt"
            sh "python3 sanity-test.py 3000 10 ${BUILD_NUMBER} stagingsanitytest.log ${PUBLIC_IP} SampleOnlineBankStaging"
            archiveArtifacts artifacts: 'stagingsanitytest.log', fingerprint: true
        }

        // lets push an event to dynatrace that indicates that we STOP a load test
        dir ('dynatrace-scripts') {
            sh './pushevent.sh SERVICE DockerService SampleOnlineBankStaging '+
               '"STOPPING Sanity Test" ${JOB_NAME} "Stopping Sanity-test of the Testing stage" '+
               '${JENKINS_URL} ${JOB_URL} ${BUILD_URL} ${GIT_COMMIT}'
        }
    }
    
    stage('ValidateStaging') {
        dir ('dynatrace-scripts') {              
            // Validate if synthetic monitor ran into any issues 
            echo "Checking Sythetic monitor status"
            try {
              STATUS = 0
                STATUS = sh 'python3 check_synthetic_run.py ${DT_URL} ${DT_TOKEN} env.execution_id'
            } catch (Exception e) {
                if (STATUS) {
                    error("Synthetic monitor has failed. Aborting the build!!")
                    currentBuild.result = 'ABORTED'
                    sh "exit ${STATUS}"
                }
            }
            
            echo "Will look for any open problems"            
            // lets see if Dynatrace AI found problems -> if so - we can stop the pipeline!
            try {
                 DYNATRACE_PROBLEM_COUNT = 0
                 DYNATRACE_PROBLEM_COUNT = sh 'python3 checkforproblems.py ${DT_URL} ${DT_TOKEN} DockerService:SampleOnlineBankStaging'
            } catch (Exception e) {
                if (DYNATRACE_PROBLEM_COUNT) {
                   error("Dynatrace opened some problem. ABORTING the build!!")
                   currentBuild.result = 'ABORTED'
                   sh "exit ${DYNATRACE_PROBLEM_COUNT}"                 
               }
            }
        }
        
        // now lets generate a report using our CLI and lets generate some direct links back to dynatrace
        dir ('dynatrace-scripts') {
            sh 'python3 make_api_call.py ${DT_URL} ${DT_TOKEN} DockerService:SampleOnlineBankStaging '+
                        'service.responsetime'
            sh 'mv Test_report.csv Test_report_staging.csv'
            archiveArtifacts artifacts: 'Test_report_staging.csv', fingerprint: true
        }
    }
    
    stage('DeployProduction') {
         // first we clean production        
        sh 'docker ps -f name=SampleOnlineBankProduction -q | xargs --no-run-if-empty docker container stop'
        sh 'docker container ls -a -fname=SampleOnlineBankProduction -q | xargs -r docker container rm'

        dir ('sample-bank-app-service') {
            // now we deploy the new container
           def app = docker.build("sample-bankapp-service:${BUILD_NUMBER}", "-f ${env.DOCKERFILE} .")
           app.run("--network mynetwork --name SampleOnlineBankProduction -p 3010:3000 "+
                "-e 'DT_CLUSTER_ID=SampleOnlineBankProduction' "+
                "-e 'DT_TAGS=Environment=Production Service=Sample-NodeJs-Service' "+
                "-e 'DT_CUSTOM_PROP=ENVIRONMENT=Production JOB_NAME=${JOB_NAME} "+
                    "BUILD_TAG=${BUILD_TAG} BUILD_NUMBER=${BUIlD_NUMBER}'")
        }
        
        dir ('dynatrace-scripts') {
            // push a deployment event on the host with the tag JenkinsInstance:
            sh './pushdeployment.sh HOST JenkinsInstance ' +
               '${BUILD_TAG} ${BUILD_NUMBER} ${JOB_NAME} ' +
                'Production SampleOnlineBankProduction'

            // Create a sample synthetic monitor so as to check the UI functionlity
           sh './synthetic-monitor.sh Production '+  '${JOB_NAME} ${BUILD_NUMBER}' + ' 3010'
            
            // Create SLOs for the staging environment
            sh "python3 create_slo.py ${DT_URL} ${DT_TOKEN} SampleOnlineBankProduction DockerService prod"
                        
          // Create a sample dashboard for the staging stage
            sh "python3 populate_slo.py ${DT_URL} ${DT_TOKEN} SampleOnlineBankProduction ${JOB_NAME} prod ${BUILD_NUMBER} DockerService"            
        }        
    }    
    
    stage('WarmUpProduction') {
        // lets push an event to dynatrace that indicates that we START a load test
        dir ('dynatrace-scripts') {
            sh './pushevent.sh SERVICE DockerService SampleOnlineBankProduction '+
               '"STARTING Load Test" ${JOB_NAME} "Starting a Load Test to warm up new prod deployment" '+
               '${JENKINS_URL} ${JOB_URL} ${BUILD_URL} ${GIT_COMMIT}'
        }
        
        // lets run some test scripts
        dir ('sample-bank-app-service-tests') {
            // start load test and run for 120 seconds - simulating traffic for Production enviornment on port 3010 
            sh "rm -f productionloadtest.log productionloadtestcontrol.txt"
            sh "python3 smoke-test.py 3010 10 ${BUILD_NUMBER} productionloadtest.log ${PUBLIC_IP} SampleOnlineBankProduction "
            archiveArtifacts artifacts: 'productionloadtest.log', fingerprint: true
        }

        // lets push an event to dynatrace that indicates that we STOP a load test
        dir ('dynatrace-scripts') {
            sh './pushevent.sh SERVICE DockerService SampleOnlineBankProduction '+
               '"STOPPING Load Test" ${JOB_NAME} "Stopping a Load Test as part of the Production warm up phase" '+
               '${JENKINS_URL} ${JOB_URL} ${BUILD_URL} ${GIT_COMMIT}'
        }
    }
    
    stage('ValidateProduction') {
        dir ('dynatrace-scripts') {      

            // lets see if Dynatrace AI found problems -> if so - we can stop the pipeline!
            try {
                 DYNATRACE_PROBLEM_COUNT = 0
                 DYNATRACE_PROBLEM_COUNT = sh 'python3 checkforproblems.py ${DT_URL} ${DT_TOKEN} DockerService:SampleOnlineBankProduction'
            } catch (Exception e) {
                if (DYNATRACE_PROBLEM_COUNT) {
                   error("Dynatrace opened some problem. ABORTING the build!!")
                   currentBuild.result = 'ABORTED'
                   sh "exit ${DYNATRACE_PROBLEM_COUNT}"                 
               }
            }
        }
        
        // now lets generate a report using our CLI and lets generate some direct links back to dynatrace
        dir ('dynatrace-scripts') {
            sh 'python3 make_api_call.py ${DT_URL} ${DT_TOKEN} DockerService:SampleOnlineBankProduction '+
                        'service.responsetime'
            sh 'mv Test_report.csv Test_report_prod.csv'
            archiveArtifacts artifacts: 'Test_report_prod.csv', fingerprint: true
        }
    }    
}
