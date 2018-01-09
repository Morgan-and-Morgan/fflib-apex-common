#!groovy
import groovy.json.JsonSlurperClassic

@NonCPS
def jsonParse(def json) {
    new groovy.json.JsonSlurperClassic().parseText(json)
}

def resolvePackageVersionId( def allPackageVersionsAvailable, def upstreamDependency ) {
    /* 
    allPackageVersionsAvailable == 
    [
        [
            Package2Id:0Ho1J0000000006SAA, 
            Branch:null, 
            MajorVersion:0, 
            MinorVersion:1, 
            PatchVersion:0, 
            BuildNumber:1, 
            Version:0.1.0.1, 
            Tag:null, 
            SubscriberPackageVersionId:04t1J0000004UCKQA2, 
            CreatedDate:2017-12-09 19:05, 
            Id:05i1J0000000006QAA, 
            NamespacePrefix:null, 
            IsBeta:true
        ],
        [
            Package2Id:0Ho1J0000000006SAA, 
            Branch:null, 
            MajorVersion:0, 
            MinorVersion:1, 
            PatchVersion:0, 
            BuildNumber:2, 
            Version:0.1.0.2, 
            Tag:null, 
            LastModifiedDate:2017-12-09 19:30, 
            Description:ver 0.1 with no ancestor with no namespace, 
            Released:false, 
            Package2Name:mm_fflib_apex_mocks, 
            Name:v0.1 trial, 
            SubscriberPackageVersionId:04t1J0000004UCPQA2, 
            CreatedDate:2017-12-09 19:30, 
            Id:05i1J000000000BQAQ, 
            NamespacePrefix:null, 
            IsBeta:true
        ]
    ]

    upstreamDependency = 
    [
        packageId:0Ho1J0000000006SAA, 
        versionNumber:0.1.0.LATEST
    ]
    */
    def result
    def workingSubscriberPackageVersionId
    def workingPackageVersionsAvailableBuildNumber
    def workingVersionNumber

    for ( packageVersionsAvailable  in allPackageVersionsAvailable ) {
        echo( "packageVersionsAvailable.Version == ${packageVersionsAvailable.Version}" )
        echo( "upstreamDependency.versionNumber == ${upstreamDependency.versionNumber}" )
        if ( upstreamDependency.packageId == packageVersionsAvailable.Package2Id ) {
            // this is the right package.  
            // is it the right version?
            if ( upstreamDependency.versionNumber.endsWith('LATEST') ) {
                def upstreamDependencyVerNumSplit = upstreamDependency.versionNumber.split('.')
                if ( packageVersionsAvailable.MajorVersion == upstreamDependencyVerNumSplit[0]
                    && packageVersionsAvailable.MinorVersion == upstreamDependencyVerNumSplit[1]
                    && packageVersionsAvailable.PatchVersion == upstreamDependencyVerNumSplit[2]
                    && packageVersionsAvailable.BuildNumber.toInteger() < workingPackageVersionsAvailableBuildNumber.toInteger()
                    ) 
                {
                    // this packageVersionsAvailable is a later build than the working version.  
                    //  Make this packageVersionsAvailable now the working
                    workingPackageVersionsAvailableBuildNumber = packageVersionsAvailable.BuildNumber
                    workingSubscriberPackageVersionId = packageVersionsAvailable.SubscriberPackageVersionId
                    workingVersionNumber = packageVersionsAvailable.Version
                }
            }
            // Look for an exact match
            else if ( upstreamDependency.versionNumber == packageVersionsAvailable.Version ) {
                echo( "version number to install is ${upstreamDependency.versionNumber}")
                result = packageVersionsAvailable.SubscriberPackageVersionId
                workingSubscriberPackageVersionId = null
                workingVersionNumber = null
                break
            }
        }
    } 
    echo ("workingSubscriberPackageVersionId out of loop = ${workingSubscriberPackageVersionId}")
    if ( workingSubscriberPackageVersionId != null ) {
        result = workingSubscriberPackageVersionId
    }
    echo ("result = ${result}")

    // the last line works as the return value
}

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
            //def sfdxProjectFileContents = readJSON(file: 'sfdx-project.json')
            def sfdxProjectFileContents = jsonParse( readFile('sfdx-project.json') )
            //echo("sfdxProjectFileContents == ${sfdxProjectFileContents}")
            SFDX_PROJECT = sfdxProjectFileContents
        }

        stage('Process Resources') {
            // if the project has upstring dependencies, install those to the scratch org first
            def packageVersionIdArray = []
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
                def allPackageVersionsAvailable = jsonSlurper.parseText(rmsg).result
                echo( "allPackageVersionsAvailable == ${allPackageVersionsAvailable}")

                def versionIdToInstall

                // loop through the SFDX_PROJECT.packageDirectories and then the packageDirectory.dependencies
                for ( packageDirectory in SFDX_PROJECT.packageDirectories ) {
                    echo("packageDirectory.dependencies == ${packageDirectory.dependencies}")
                    for ( upstreamDependency in packageDirectory.dependencies ) {
                        echo( "upstreamDependency == ${upstreamDependency} -- should be installed to test org")

                        versionIdToInstall = resolvePackageVersionId( allPackageVersionsAvailable, upstreamDependency )


                        
                    }
                }

                // if the upstreamDependency.versionNumber contains the word "LATEST", then use the latest version for that <major>.<minor>.<patch> 
                // else use the version that matches the <major>.<minor>.<patch> 

            } else {
                echo( "no upstream dependencies found for this project" )
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

        stage('Post Build Notifications') {
            slackSend channel: '#sf-ci-alerts', failOnError: true, message: 'started ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)', tokenCredentialId: 'Slack-Integration-Token-SF-CI-ALERTS'
        }
    }
}