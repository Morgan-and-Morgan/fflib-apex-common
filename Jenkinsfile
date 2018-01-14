#!groovy
import groovy.json.JsonSlurperClassic

@NonCPS
def jsonParse(def json) {
    new groovy.json.JsonSlurperClassic().parseText(json)
}

@NonCPS
def recordPackageVersionArtifact( def artifactDirectory, def packageVersion ) {
    def fileToFingerprint = "${artifactDirectory}/package2-${packageVersion.Package2Id}-version-${packageVersion.MajorVersion}.${packageVersion.MinorVersion}.${packageVersion.PatchVersion}.${packageVersion.BuildNumber}-branch-${packageVersion.Branch}-${packageVersion.SubscriberPackageVersionId}.packageVersion"
    echo("creating package version artifact for ${fileToFingerprint}")

    writeFile file: fileToFingerprint, text: "${packageVersion}"
}

@NonCPS
def resolvePackageVersion( def allPackageVersionsAvailable, def upstreamDependency ) {

    def result
    def workingPackageVersion
    def workingPackageVersionsAvailableBuildNumber = 0

    for ( packageVersionAvailable  in allPackageVersionsAvailable ) {

        if ( upstreamDependency.packageId == packageVersionAvailable.Package2Id ) {
            // this is the right package.  
            // is it the right version?
            if ( upstreamDependency.versionNumber.endsWith('LATEST') ) {

                def upstreamDependencyVerNumSplit = upstreamDependency.versionNumber.split("\\.") // need to escape the dot to get a literal period

                echo( "packageVersionAvailable.Version == ${packageVersionAvailable.Version}" )
                echo( "upstreamDependency.versionNumber == ${upstreamDependency.versionNumber}" )
                echo( "packageVersionAvailable.MajorVersion == ${packageVersionAvailable.MajorVersion}" )
                echo( "upstreamDependencyVerNumSplit[0] == ${upstreamDependencyVerNumSplit[0]}")
                echo( "packageVersionAvailable.MinorVersion == ${packageVersionAvailable.MinorVersion}" )
                echo( "upstreamDependencyVerNumSplit[1] == ${upstreamDependencyVerNumSplit[1]}")
                echo( "packageVersionAvailable.PatchVersion == ${packageVersionAvailable.PatchVersion}" )
                echo( "upstreamDependencyVerNumSplit[2] == ${upstreamDependencyVerNumSplit[2]}")
                echo( "workingPackageVersionsAvailableBuildNumber == ${workingPackageVersionsAvailableBuildNumber}")
                echo( "packageVersionAvailable.BuildNumber == ${packageVersionAvailable.BuildNumber}")

                if ( packageVersionAvailable.MajorVersion == upstreamDependencyVerNumSplit[0].toInteger()
                    && packageVersionAvailable.MinorVersion == upstreamDependencyVerNumSplit[1].toInteger()
                    && packageVersionAvailable.PatchVersion == upstreamDependencyVerNumSplit[2].toInteger()
                    && packageVersionAvailable.BuildNumber > workingPackageVersionsAvailableBuildNumber
                    ) 
                {
                    echo( 'point 2A' )

                    // this packageVersionAvailable is a later build than the working version.  
                    //  Make this packageVersionAvailable now the working
                    workingPackageVersionsAvailableBuildNumber = packageVersionAvailable.BuildNumber
                    workingPackageVersion = packageVersionAvailable
                }
            }
            // Look for an exact match
            else if ( upstreamDependency.versionNumber == packageVersionAvailable.Version ) {
                echo( "version number to install is ${upstreamDependency.versionNumber}")

                result = packageVersionAvailable
                workingPackageVersion = null
                break
            }
        }
    } 
    echo ("workingPackageVersion out of loop = ${workingPackageVersion}")
    if ( workingPackageVersion != null ) {
        result = workingPackageVersion
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
            def package2IdArray = []
            for ( packageDirectory in SFDX_PROJECT.packageDirectories ) {

                echo( "packageDirectory ==  ${packageDirectory}" )
                
                for ( upstreamDependency in packageDirectory.dependencies ) {
                    echo( "upstreamDependency == ${upstreamDependency} -- installing to test org")
                    package2IdArray.add( upstreamDependency.packageId )
                }
            }

            if ( ! package2IdArray.isEmpty() ) {

                //def allPackageVersionsAvailable = findAllPackageVersionsAvailableByPackageId( package2IdArray.join(','), toolbelt )
                echo("finding all package versions for package ids found")
                rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:package2:version:list --package2ids ${package2IdArray.join(',')} --json "
                printf rmsg
                
                def allPackageVersionsAvailable = jsonParse(rmsg).result

                def packageVersion

                // loop through the SFDX_PROJECT.packageDirectories and then the packageDirectory.dependencies
                for ( packageDirectory in SFDX_PROJECT.packageDirectories ) {
                    echo("packageDirectory.dependencies == ${packageDirectory.dependencies}")
                    for ( upstreamDependency in packageDirectory.dependencies ) {
                        echo( "upstreamDependency == ${upstreamDependency} -- should be installed to test org")

                        //versionIdToInstall = resolvePackageVersionId( allPackageVersionsAvailable, upstreamDependency )
                        packageVersion = resolvePackageVersion( allPackageVersionsAvailable, upstreamDependency )

                        //echo("versionIdToInstall == ${versionIdToInstall}")
                        echo("packageVersion == ${packageVersion}")

                        //if ( versionIdToInstall != null ) {
                        if ( packageVersion != null ) {
                            //echo ("installing version ${versionIdToInstall}")
                            echo("installing version ${packageVersion.Version}")

                            recordPackageVersionArtifact ( RUN_ARTIFACT_DIR, packageVersion )

                            //rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:package:install --id ${versionIdToInstall} --json --wait 3 "
                            rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:package:install --id ${packageVersion.SubscriberPackageVersionId} --json --wait 3 "
                            
                            printf rmsg
//TODO : Need to setup the check to ensure that package version has been installed.
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
        }

        stage('Compile') {
            echo("Push To Test Org And Compile")
            rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:source:push --targetusername ${SFDC_USERNAME}"
            if (rc != 0) {
                error 'push failed'
            }
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
                                    SFDX_NEW_PACKAGE_VERSION_ID = packageVersionCreationCheckResponseResult.Package2VersionId
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
                    echo("packageVersionCreationResponse == ${packageVersionCreationResponse}")
                    SFDX_NEW_PACKAGE_VERSION_ID = packageVersionCreationResponse.result.Package2VersionId
                }
            }
            echo( "SFDX_NEW_PACKAGE_VERSION_ID == ${SFDX_NEW_PACKAGE_VERSION_ID}")
        }

        stage('Install') {
            // this will be where the fingerprints of the build are created and then stored in Jenkins
            if ( SFDX_NEW_PACKAGE_VERSION_ID != null ) {
                // then a package was created.  Record its finger prints
                //def allPackageVersionsAvailable = findAllPackageVersionsAvailableByPackageId( SFDX_NEW_PACKAGE_ID, toolbelt )
                echo("finding all package versions for package ids found")
                rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:package2:version:list --package2ids ${SFDX_NEW_PACKAGE_ID} --json "
                printf rmsg
                
                def allPackageVersionsAvailable = jsonParse(rmsg).result

                // loop through all allPackageVersionsAvailable until you find the new one with the SFDX_NEW_PACKAGE_VERSION_ID
                for ( packageVersionAvailable in allPackageVersionsAvailable ) {
                    echo ("packageVersionAvailable == ${packageVersionAvailable}")
                    echo ("SFDX_NEW_PACKAGE_ID == ${SFDX_NEW_PACKAGE_ID}")
                    echo ("packageVersionAvailable.Package2Id == ${packageVersionAvailable.Package2Id}")
                    echo ("SFDX_NEW_PACKAGE_VERSION_ID == ${SFDX_NEW_PACKAGE_VERSION_ID}")
                    echo ("packageVersionAvailable.Id == ${packageVersionAvailable.id}")
                    if ( SFDX_NEW_PACKAGE_ID == packageVersionAvailable.Package2Id && SFDX_NEW_PACKAGE_VERSION_ID == packageVersionAvailable.Id) {
                        echo ("found a match")
                        recordPackageVersionArtifact( RUN_ARTIFACT_DIR, packageVersionAvailable )
                        break
                    }
                }
            }
            
            fingerprint "${RUN_ARTIFACT_DIR}/*.packageVersion"

        }

        stage('Clean') {
            echo('Deleting scratch org')
            rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:org:delete --targetusername ${SFDC_USERNAME} --noprompt"
            if (rc != 0) { 
                error "deletion of scratch org ${HUB_ORG} failed"
            }

//            cleanWs cleanWhenAborted: false, cleanWhenFailure: false, cleanWhenNotBuilt: false, cleanWhenUnstable: false, notFailBuild: true
        }

        stage('Post Build Notifications') {
            slackSend channel: '#sf-ci-alerts', color: 'good', failOnError: true, message: "build completed ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)", tokenCredentialId: 'Slack-Integration-Token-SF-CI-ALERTS'
        }
    }
}