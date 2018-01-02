#!groovy
import groovy.json.JsonSlurperClassic
node {

    def BUILD_NUMBER=env.BUILD_NUMBER
    def RUN_ARTIFACT_DIR="target/${BUILD_NUMBER}"
    def SFDC_USERNAME
    def SFDX_PROJECT

    def HUB_ORG=env.HUB_ORG_DH
    def SFDC_HOST = env.SFDC_HOST_DH
    def JWT_KEY_CRED_ID = env.JWT_CRED_ID_DH
    def CONNECTED_APP_CONSUMER_KEY=env.CONNECTED_APP_CONSUMER_KEY_DH

    //def toolbelt = tool 'toolbelt'
    def toolbelt = tool 'sfdx-cli-toolbelt'

    stage('Checkout Source') {
        // when running in multi-branch job, one must issue this command
        checkout scm
    }

    stage('Validate') {
        // if sfdx-project.json file is not present, then abort build
        def sfdxProjectFileExists = fileExists 'sfdx-project.json'
        if ( ! sfdxProjectFileExists ) {
            error 'SFDX project file (sfdx-project.json) not found.'
        }
    }

    withCredentials([file(credentialsId: JWT_KEY_CRED_ID, variable: 'jwt_key_file')]) {

        stage('Initialize') {
            
            echo("Authenticate To Dev Hub...")
            rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:auth:jwt:grant --clientid ${CONNECTED_APP_CONSUMER_KEY} --username ${HUB_ORG} --jwtkeyfile ${jwt_key_file} --setdefaultdevhubusername --instanceurl ${SFDC_HOST}"
            if (rc != 0) { error 'hub org authorization failed' }

            echo("Create Scratch Org...")
            rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:org:create --definitionfile config/project-scratch-def.json --json --setdefaultusername"
            printf rmsg

            // need to pull out assigned username from the scratch org that was just generated
            echo('Deserialize the force:org:create response')
            def jsonSlurper = new JsonSlurperClassic()
            def robj = jsonSlurper.parseText(rmsg)
            if (robj.status != 0) { error 'org creation failed: ' + robj.message }
            SFDC_USERNAME=robj.result.username
            robj = null

            // process the sfdx-project.json file for later user
            echo('Deserialize the sfdx-project.json ')
            def sfdxProjectFileContents = readFile file: 'sfdx-project.json'
            echo("sfdxProjectFileContents == ${sfdxProjectFileContents}")
            SFDX_PROJECT = jsonSlurper.parseText( sfdxProjectFileContents )
        }

        stage('Process Resources') {
            // if the project has upstring dependencies, install those to the scratch org first
            def packageVersionIdArray = String[]
            for ( packageDirectory in SFDX_PROJECT.packageDirectories ) {

                echo( "packageDirectory ==  ${packageDirectory}" )
                
                for ( upstreamDependency in packageDirectory.dependencies ) {
                    echo( "upstreamDependency == ${upstreamDependency} -- installing to test org")
                    packageVersionIdArray.add( upstreamDependency.packageId )
                }
            }

            if ( ! packageVersionIdArray.isEmpty() ) {
                echo("finding all package versions for package ids found")
                rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:package2:version:list --package2ids ${packageVersionIdArray.join(',')} --json "
                printf rmsg
                
                def jsonSlurper = new JsonSlurperClassic()
                def allPackageVersionsAvailable = jsonSlurper.parseText(rmsg)
                echo( "allPackageVersionsAvailable == ${allPackageVersionsAvailable}")
            } else {
                echo( "upstream dependencies found for this project" )
            }
            

                    //rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:auth:jwt:grant --clientid ${CONNECTED_APP_CONSUMER_KEY} --username ${HUB_ORG} --jwtkeyfile ${jwt_key_file} --setdefaultdevhubusername --instanceurl ${SFDC_HOST}"
                    //rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:package:install -i 04t1J0000004UBgQAM --targetusername 
                    //if (rc != 0) { error 'hub org authorization failed' }

        }

        stage('Compile') {
            echo("Push To Test Org And Compile")
            rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:source:push --targetusername ${SFDC_USERNAME}"
            if (rc != 0) {
                error 'push failed'
            }
            // assign permset
//            rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:user:permset:assign --targetusername ${SFDC_USERNAME} --permsetname DreamHouse"
//            if (rc != 0) {
//                error 'permset:assign failed'
//            }
        }

        stage('Generate Test Sources') {
            // generate mock files, if needed
                // a mock file is present
                // expression { mock file is present }
                // generate mock file here
        }

        stage('Test') {
            echo( 'Run All Local Apex Tests' )
            sh "mkdir -p ${RUN_ARTIFACT_DIR}"
            timeout(time: 120, unit: 'SECONDS') {
                rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:apex:test:run --testlevel RunLocalTests --outputdir ${RUN_ARTIFACT_DIR} --resultformat tap --targetusername ${SFDC_USERNAME}"
                if (rc != 0) {
                    error 'apex test run failed'
                }

                // Process all unit test reports
                echo( "Collect All Test Results")
                junit keepLongStdio: true, testResults: "${RUN_ARTIFACT_DIR}/**/*-junit.xml"
            }
        }

        stage('Clean') {
            echo('Deleting scratch org')
            rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:auth:jwt:grant --clientid ${CONNECTED_APP_CONSUMER_KEY} --username ${HUB_ORG} --jwtkeyfile ${jwt_key_file} --setdefaultdevhubusername --instanceurl ${SFDC_HOST}"
            if (rc != 0) { 
                error "deletion of scratch org ${HUB_ORG} failed"
            }
        }
    }
}