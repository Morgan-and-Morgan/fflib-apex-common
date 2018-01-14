#!groovy
import groovy.json.JsonSlurperClassic

@NonCPS
def jsonParse(def json) {
    new groovy.json.JsonSlurperClassic().parseText(json)
}

@NonCPS
def recordPackageVersionFingerprint( def artifactDirectory, def packageId, def packageVersionId ) {
    def fileToFingerprint = "${artifactDirectory}/package2-${packageId}-version-${packageVersionId}.packageVersion"
    echo("recording fingerprint for ${fileToFingerprint}")

    writeFile file: fileToFingerprint, text: "package2-${packageId}-version-${packageVersionId}"
    //fingerprint fileToFingerprint`
    archiveArtifacts allowEmptyArchive: true, artifacts: "${fileToFingerprint}", fingerprint: true
}

@NonCPS
def resolvePackageVersionId( def allPackageVersionsAvailable, def upstreamDependency ) {

    def result
    def workingSubscriberPackageVersionId
    def workingPackageVersionsAvailableBuildNumber = 0
    def workingVersionNumber

    for ( packageVersionsAvailable  in allPackageVersionsAvailable ) {

        if ( upstreamDependency.packageId == packageVersionsAvailable.Package2Id ) {
            // this is the right package.  
            // is it the right version?
            if ( upstreamDependency.versionNumber.endsWith('LATEST') ) {

                def upstreamDependencyVerNumSplit = upstreamDependency.versionNumber.split("\\.") // need to escape the dot to get a literal period

                echo( "packageVersionsAvailable.Version == ${packageVersionsAvailable.Version}" )
                echo( "upstreamDependency.versionNumber == ${upstreamDependency.versionNumber}" )
                echo( "packageVersionsAvailable.MajorVersion == ${packageVersionsAvailable.MajorVersion}" )
                echo( "upstreamDependencyVerNumSplit[0] == ${upstreamDependencyVerNumSplit[0]}")
                echo( "packageVersionsAvailable.MinorVersion == ${packageVersionsAvailable.MinorVersion}" )
                echo( "upstreamDependencyVerNumSplit[1] == ${upstreamDependencyVerNumSplit[1]}")
                echo( "packageVersionsAvailable.PatchVersion == ${packageVersionsAvailable.PatchVersion}" )
                echo( "upstreamDependencyVerNumSplit[2] == ${upstreamDependencyVerNumSplit[2]}")
                echo( "workingPackageVersionsAvailableBuildNumber == ${workingPackageVersionsAvailableBuildNumber}")
                echo( "packageVersionsAvailable.BuildNumber == ${packageVersionsAvailable.BuildNumber}")

                if ( packageVersionsAvailable.MajorVersion == upstreamDependencyVerNumSplit[0].toInteger()
                    && packageVersionsAvailable.MinorVersion == upstreamDependencyVerNumSplit[1].toInteger()
                    && packageVersionsAvailable.PatchVersion == upstreamDependencyVerNumSplit[2].toInteger()
                    && packageVersionsAvailable.BuildNumber > workingPackageVersionsAvailableBuildNumber
                    ) 
                {
                    echo( 'point 2A' )

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

                //recordPackageVersionFingerprint ( RUN_ARTIFACT_DIR, packageVersionsAvailable.Package2Id, packageVersionsAvailable.SubscriberPackageVersionId )

                result = packageVersionsAvailable.SubscriberPackageVersionId
                workingSubscriberPackageVersionId = null
                workingVersionNumber = null
                break
            }
        }
    } 
    echo ("workingSubscriberPackageVersionId out of loop = ${workingSubscriberPackageVersionId}")
    if ( workingSubscriberPackageVersionId != null ) {
        //recordPackageVersionFingerprint ( RUN_ARTIFACT_DIR, packageVersionsAvailable.Package2Id, workingSubscriberPackageVersionId )
        result = workingSubscriberPackageVersionId
    }
    echo ("result = ${result}")

    // the last line works as the return value
    return result
}

node {

    def BUILD_NUMBER=env.BUILD_NUMBER
    def RUN_ARTIFACT_DIR="target/${BUILD_NUMBER}"
    def SFDC_USERNAME
    def SFDX_PROJECT
    def SFDX_NEW_PACKAGE_ID
    def SFDX_NEW_PACKAGE_VERSION_ID

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
            jsonSlurper = null 

            // process the sfdx-project.json file for later user
            echo('Deserialize the sfdx-project.json ')
            
            def sfdxProjectFileContents = jsonParse( readFile('sfdx-project.json') )
            
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
                jsonSlurper = null 

                def versionIdToInstall

                // loop through the SFDX_PROJECT.packageDirectories and then the packageDirectory.dependencies
                for ( packageDirectory in SFDX_PROJECT.packageDirectories ) {
                    echo("packageDirectory.dependencies == ${packageDirectory.dependencies}")
                    for ( upstreamDependency in packageDirectory.dependencies ) {
                        echo( "upstreamDependency == ${upstreamDependency} -- should be installed to test org")

                        versionIdToInstall = resolvePackageVersionId( allPackageVersionsAvailable, upstreamDependency )

                        echo("versionIdToInstall == ${versionIdToInstall}")

                        if ( versionIdToInstall != null ) {
                            echo ("installing version ${versionIdToInstall}")

                            recordPackageVersionFingerprint ( RUN_ARTIFACT_DIR, upstreamDependency.packageId, versionIdToInstall )

                            //rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:package:install -i ${versionIdToInstall}"
                            //echo ("rc == ${rc}")
                            //if (rc != 0) { error "installtion of package version ${versionIdToInstall} failed" }
                            
                            rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:package:install --id ${versionIdToInstall} --json --wait 3 "
                            printf rmsg

                            echo ("point 3A")
                            echo ( rmsg )
                            echo ("point 3B")
//                            def installationRequest = jsonSlurper.parseText(rmsg).result
//
//                            rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:package:install:get -id ${versionIdToInstall} --json --wait 3 "
//                            printf rmsg



                        }
                        else {
                            echo ("No package version found to install for ")
                        }
                        
                        
                    }
                }

            } else {
                echo( "no upstream dependencies found for this project" )
            }

            allPackageVersionsAvailable = null 
//            jsonSlurper = null 
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

        stage('prepare-package') {
            // this is where the package version will be organized
        }

        stage('package') {
            // this is where the package version will be created

            def directoryToUseForPackageInstall 


            // What is the default package and what is its directory?
            for ( packageDirectory in SFDX_PROJECT.packageDirectories ) {
                if ( packageDirectory.default ) {
                    directoryToUseForPackageInstall = packageDirectory.path 
                    SFDX_NEW_PACKAGE_ID = packageDirectory.id 
                    break 
                }
            }

            if ( directoryToUseForPackageInstall == null ) {
                error 'unable to determine the package directory for package creation'
            }

            //rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:package2:version:create --directory ${directoryToUseForPackageInstall} --json --wait 2"
            rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:package2:version:create --directory ${directoryToUseForPackageInstall} --json"
            printf rmsg

            def packageVersionCreationResponse = jsonParse(rmsg)

            echo ("packageVersionCreationResponse == ${packageVersionCreationResponse}")

            if ( packageVersionCreationResponse.status == 0 ) {

                if( packageVersionCreationResponse.result.Status == 'InProgress' 
                    || packageVersionCreationResponse.result.Status == 'Queued') {
                    // The package version creation is still underway

                    timeout(15) {
                        waitUntil {
                            script {
                                // use the packageVersionCreationResponse.result.Id for this command verses SFDX_NEW_PACKAGE_VERSION_ID because we are yet
                                //  certain that the package was created correctly
                                rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:package2:version:create:get --package2createrequestid ${packageVersionCreationResponse.result.Id} --json"
                                printf rmsg

                                def jsonSlurper2 = new JsonSlurperClassic()
                
                                //def packageVersionCreationCheckResponse = jsonParse(rmsg) 
                                def packageVersionCreationCheckResponse = jsonSlurper2.parseText(rmsg)

                                // The JSON "result" is currently an array.  That is a potential bug. Refer to Salesforce DX Success Community post for details https://success.salesforce.com/0D53A00003OTsAD
                                def packageVersionCreationCheckResponseResult = jsonSlurper2.parseText(rmsg).result[0]
                                
                                echo ("packageVersionCreationCheckResponse == ${packageVersionCreationCheckResponse}")
                                //echo ("packageVersionCreationCheckResponseResult == ${packageVersionCreationCheckResponseResult}")
                                echo("marker 3A")
                                echo (packageVersionCreationCheckResponseResult.Status)
                                echo("marker 3A2")

                                if ( packageVersionCreationCheckResponseResult.Status == "Success" ) {
                                    SFDX_NEW_PACKAGE_VERSION_ID = packageVersionCreationCheckResponseResult.Id
                                }

                                echo("marker 3B")

                                def isPackageVersionCreationCompleted = packageVersionCreationCheckResponse.status == 0 && packageVersionCreationCheckResponseResult.Status != "InProgress" && packageVersionCreationCheckResponseResult.Status != "Queued"
                                echo( "isPackageVersionCreationCompleted == ${isPackageVersionCreationCompleted}")
                                return isPackageVersionCreationCompleted
                            }
                        }
                        echo("Exited the waitUntil phase")
                    }
                    echo("Exited the timeout phase")
                }
                else if( packageVersionCreationResponse.result.Status == 'Success' ) {
                    SFDX_NEW_PACKAGE_VERSION_ID = packageVersionCreationResponse.result.Id
                }
            }
            echo( "SFDX_NEW_PACKAGE_VERSION_ID == ${SFDX_NEW_PACKAGE_VERSION_ID}")
        }

        stage('Install') {
            // this will be where the fingerprints of the build are created and then stored in Jenkins
            if ( SFDX_NEW_PACKAGE_VERSION_ID != null ) {
                // then a package was created.  Record its finger prints
                recordPackageVersionFingerprint( RUN_ARTIFACT_DIR, SFDX_NEW_PACKAGE_ID, SFDX_NEW_PACKAGE_VERSION_ID )
            }


        }

        stage('Clean') {
            echo('Deleting scratch org')
            rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:org:delete --targetusername ${SFDC_USERNAME} --noprompt"
            if (rc != 0) { 
                error "deletion of scratch org ${HUB_ORG} failed"
            }
        }

        stage('Post Build Notifications') {
            slackSend channel: '#sf-ci-alerts', color: 'good', failOnError: true, message: "build completed ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)", tokenCredentialId: 'Slack-Integration-Token-SF-CI-ALERTS'
        }
    }
}