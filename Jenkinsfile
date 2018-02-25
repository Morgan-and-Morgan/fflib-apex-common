#!groovy
import groovy.json.JsonSlurperClassic

/*
 * @version 0.3.0.SNAPSHOT
 *
 *  Change log:
 *  - commented out the "printf" commands as there was a potential issue that an illegal character was trying to be published
 * 
 *  Backlog:
 *  - Potential processing issue on Jenkins where source:push throws error
 */

pipeline {
    agent any 
    environment {
        def BUILD_NUMBER = "${env.BUILD_NUMBER}"
        def RUN_ARTIFACT_DIR = "target/${BUILD_NUMBER}"
        def SFDC_USERNAME = ""
        def SFDX_PROJECT = ""
        def SFDX_NEW_PACKAGE_ID = ""
        def SFDX_NEW_PACKAGE_VERSION_ID = ""

        def HUB_ORG = "${env.HUB_ORG_DH}"
        def SFDC_HOST = "${env.SFDC_HOST_DH}"
        def JWT_KEY_CRED_ID = "${env.JWT_CRED_ID_DH}"
        def CONNECTED_APP_CONSUMER_KEY = "${env.CONNECTED_APP_CONSUMER_KEY_DH}"

        // Tools 
        def toolbelt = tool 'sfdx-cli-toolbelt'
        def pmdtoolbelt = tool 'pmd-toolbelt'
    }

//    tools {
//        // for the moment, it appears that defining the custom tool in the declarative tools section is not supported.  
//        //  ref -- https://jenkins.io/doc/book/pipeline/syntax/#tools
//        //  ref -- https://wiki.jenkins.io/display/JENKINS/Custom+Tools+Plugin?focusedCommentId=135470277#comment-135470277
//        //def toolbelt = tool 'sfdx-cli-toolbelt'
//        //def pmdtoolbelt = tool 'pmd-toolbelt'
//        //com.cloudbees.jenkins.plugins.customtools.CustomTool 'sfdx-cli-toolbelt'
//        // com.cloudbees.jenkins.plugins.customtools.CustomTool sfdx-cli-toolbelt
//        //com.cloudbees.jenkins.plugins.customtools.CustomTool 'pmd-toolbelt'   
//    }


    triggers {
        //cron('H */4 * * 1-5')
        // TODO: Figure out how to set this up!!!!!
        //upstream(upstreamProjects: 'job1,job2', threshold: hudson.model.Result.SUCCESS)
        //upstream(upstreamProjects: "some_project/some_branch", threshold: hudson.model.Result.SUCCESS)
        //upstream(upstreamProjects: "some_project/" + env.BRANCH_NAME.replaceAll("/", "%2F"), threshold: hudson.model.Result.SUCCESS)

        //working attempts
        upstream( upstreamProjects: "Salesforce-DX-Projects/fflib-apex-mocks/mm-master-sfdx", threshold: hudson.model.Result.SUCCESS )
    }

    stages {
//        stage('Checkout Source') {
//            // when running in multi-branch job, one must issue this command
//            steps {
//                checkout scm
//            }
//        }

        stage('Validate') {
            steps {
                script {
                    // if sfdx-project.json file is not present, then abort build
                    def sfdxProjectFileExists = fileExists 'sfdx-project.json'
                    if ( ! sfdxProjectFileExists ) {
                        error 'SFDX project file (sfdx-project.json) not found.'
                    }
                }
            }
        }

        stage('Initialize') {
            steps {
                script{
                    sh "mkdir -p ${RUN_ARTIFACT_DIR}"
                }
                    
                echo("Authenticate To Dev Hub...")
                withCredentials( [ file( credentialsId: JWT_KEY_CRED_ID, variable: 'jwt_key_file') ] ) {
                    script {
                        rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:auth:jwt:grant --clientid ${CONNECTED_APP_CONSUMER_KEY} --username ${HUB_ORG} --jwtkeyfile ${jwt_key_file} --setdefaultdevhubusername --instanceurl ${SFDC_HOST}"
                        if (rc != 0) { handleError('hub org authorization failed') }
                    }
                    echo("Create Scratch Org...")
                    script {
                        rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:org:create --definitionfile config/project-scratch-def.json --json --setdefaultusername"

                        // need to pull out assigned username from the scratch org that was just generated
                        echo('Deserialize the force:org:create response')

                        //def jsonSlurper = new JsonSlurperClassic()
                        //def robj = jsonSlurper.parseText(rmsg)
                        def robj = jsonParse( rmsg )
                        
                        //if (robj.status != 0) { handleError('org creation failed: ' + robj.message) }
                        if (robj.status != 0) { error('org creation failed: ' + robj.message) }
                        
                        SFDC_USERNAME=robj.result.username
                        robj = null
                        jsonSlurper = null 
                    }

                    // process the sfdx-project.json file for later user
                    echo('Deserialize the sfdx-project.json ')
                    
                    script {
                        def sfdxProjectFileContents = jsonParse( readFile('sfdx-project.json') )                
                        SFDX_PROJECT = sfdxProjectFileContents
                    }
                }
            }
        }

        stage('Process Resources') {
            steps {
                // if the project has upstring dependencies, install those to the scratch org first

                // Loop through the dependencies.  Find out which ones need to be resolved
                // The order that the dependencies are listed in the SFDX_PROJECT.packageDirectories array
                //  is important because that will be the order that they are installed in.  If one dependency
                //  needs another one listed to be installed first, then place that dependency earlier in the list
                //  of the SFDX_PROJECT.packageDirectories before the dependent artifact.

                echo("finding all package versions available")
                
                script {
                    rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:package2:version:list  --json "
                    def allPackageVersionsAvailable = jsonParse(rmsg).result
                    def packageVersion

                    // loop through the SFDX_PROJECT.packageDirectories and then the packageDirectory.dependencies
                    for ( packageDirectory in SFDX_PROJECT.packageDirectories ) {
                        echo("packageDirectory.dependencies == ${packageDirectory.dependencies}")

                        for ( upstreamDependency in packageDirectory.dependencies ) {
                            echo( "upstreamDependency == ${upstreamDependency} -- should be installed to test org")

                            packageVersion = resolvePackageVersion( allPackageVersionsAvailable, upstreamDependency )

                            echo("packageVersion == ${packageVersion}")

                            if ( packageVersion != null ) {
                                echo("installing version ${packageVersion.Version}")

                                recordPackageVersionArtifact ( RUN_ARTIFACT_DIR, packageVersion )

                                rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:package:install --id ${packageVersion.SubscriberPackageVersionId} --json --wait 3 "
                                def response = jsonParse(rmsg)
                                echo("package install response == ${response}")

                                if ( response.status == 1 && response.message.startsWith( 'The package version is not fully available' ) ) {
                                    echo( "waiting on the upstream package to complete creation")
                                    sleep time: 3, unit: 'MINUTES'
                                    // try the install again.
                                    rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:package:install --id ${packageVersion.SubscriberPackageVersionId} --json --wait 3 "
                                    response = jsonParse(rmsg)
                                }
                                // 
                                //TODO : Need to setup the check to ensure that package version has been installed.
                                //                          handleError()
                                //                            def installationRequest = jsonSlurper.parseText(rmsg).result
                                //
                                //                            rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:package:install:get -id ${versionIdToInstall} --json --wait 3 "
                                //                            printf rmsg
                            }
                            else {
                                echo ("No package version found to install")
                            }
                        }
                    }

                    allPackageVersionsAvailable = null 
                }
            }
        }

        stage('Compile') {
            steps {
                script {
                    echo("Push To Test Org And Compile")
                    rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:source:push --targetusername ${SFDC_USERNAME} --json"
                    //printf rmsg

                    def response = jsonParse( rmsg )

                    if (response.status != 0) {
                        echo(response)
                        handleError("push failed -- ${response.message}")
                    }
                }
            }
        }

        stage('Test') {
            failFast true
            parallel {
                stage('Test: Apex Unit Tests') {
                    steps {
                        echo( 'Run All Local Apex Tests' )
                        timeout(time: 120, unit: 'SECONDS') {
                            script {
                                rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:apex:test:run --testlevel RunLocalTests --outputdir ${RUN_ARTIFACT_DIR} --resultformat tap --targetusername ${SFDC_USERNAME} --json"
                                def response = jsonParse( rmsg )
                                if (response.status != 0) {
                                    echo(response)
                                    handleError("apex test run failed -- ${response.message}")
                                }

                                // Process all unit test reports
                                echo( "Collect All Test Results")
                                junit keepLongStdio: true, testResults: "${RUN_ARTIFACT_DIR}/**/*-junit.xml"
                            }
                        }
                    }
                }
                stage('Test: PMD Static Analysis') {
                    steps {
                        echo( 'Analyze code via PMD' )
                        timeout(time: 120, unit: 'SECONDS') {
                            script {
                                rmsg = sh returnStdout: true, script: "${pmdtoolbelt}/pmd-bin-6.0.1/bin/run.sh pmd -d sfdx-source -l apex -r ${RUN_ARTIFACT_DIR}/pmd_report.xml -f xml -rulesets category/apex/errorprone.xml -failOnViolation false"
                                echo( rmsg )
                                pmd canComputeNew: false, defaultEncoding: '', healthy: '25%', pattern: "${RUN_ARTIFACT_DIR}/pmd_report.xml", unHealthy: '10%'
                            }
                        }
                    }
                }
            }
        }

        stage('package') {
            steps {
                script {
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

                    if ( SFDX_NEW_PACKAGE_ID == null ) {
                        handleError( "unable to determine SFDX_NEW_PACKAGE_ID in stage:package")
                    }

                    if ( directoryToUseForPackageInstall == null ) {
                        handleError( "unable to determine the package directory for package creation in stage:package")
                    }

                    def commandScriptString = "${toolbelt}/sfdx force:package2:version:create --directory ${directoryToUseForPackageInstall} --json --tag ${env.BUILD_TAG.replaceAll(' ','-')}"

                    if ( env.BRANCH_NAME != null ) {
                        commandScriptString = commandScriptString + " --branch ${env.BRANCH_NAME}"
                    }
                    echo ("commandScriptString == ${commandScriptString}")

                    rmsg = sh returnStdout: true, script: commandScriptString
                    //printf rmsg

                    def packageVersionCreationResponse = jsonParse(rmsg)

                    echo ("packageVersionCreationResponse == ${packageVersionCreationResponse}")

                    if ( packageVersionCreationResponse.status != 0 ) {
                        echo( packageVersionCreationResponse )
                        handleError( "package version creation has failed -- ${packageVersionCreationResponse.message}")
                    }

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
                                        //printf rmsg

                                        def jsonSlurper2 = new JsonSlurperClassic()
                        
                                        //def packageVersionCreationCheckResponse = jsonParse(rmsg) 
                                        def packageVersionCreationCheckResponse = jsonSlurper2.parseText(rmsg)

                                        echo ("packageVersionCreationCheckResponse == ${packageVersionCreationCheckResponse}")

                                        if ( packageVersionCreationCheckResponse.status != 0 ) {
                                            handleError( "force:package2:version:create:get failed -- ${packageVersionCreationCheckResponse.message}")
                                        }

                                        // The JSON "result" is currently an array.  That is a SFDX bug -- W-4621618
                                        // Refer to Salesforce DX Success Community post for details https://success.salesforce.com/0D53A00003OTsAD
                                        def packageVersionCreationCheckResponseResult = jsonSlurper2.parseText(rmsg).result[0]
                                        
                                        if ( packageVersionCreationCheckResponseResult.Status == 'Error' ) {
                                            handleError( "force:package2:version:create:get failed -- ${packageVersionCreationCheckResponseResult.message}")
                                        }
                                        else if ( packageVersionCreationCheckResponseResult.Status == "Success" ) {
                                            SFDX_NEW_PACKAGE_VERSION_ID = packageVersionCreationCheckResponseResult.Package2VersionId
                                        }

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
            }
        }

        stage('Install') {
            steps {
                script {
                    // this will be where the fingerprints of the build are created and then stored in Jenkins
                    if ( SFDX_NEW_PACKAGE_VERSION_ID != null ) {

                        // then a package was created.  Record its finger prints
                        echo("finding all package versions for package ids found")
                        rmsg = sh returnStdout: true, script: "${toolbelt}/sfdx force:package2:version:list --package2ids ${SFDX_NEW_PACKAGE_ID} --json "
                        //printf rmsg

                        def response = jsonParse( rmsg )
                        
                        def allPackageVersionsAvailable = response.result

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
                    
                    archiveArtifacts artifacts: "${RUN_ARTIFACT_DIR}/*.packageVersion", fingerprint: true, onlyIfSuccessful: true
                }
            }
        }
    }

    post {
        always {
            echo('Deleting scratch org')
            script {
                rc = sh returnStatus: true, script: "${toolbelt}/sfdx force:org:delete --targetusername ${SFDC_USERNAME} --noprompt"
                if (rc != 0) { 
                    error "deletion of scratch org ${HUB_ORG} failed"
                }
            }
        }
        success {
            slackSend channel: '#sf-ci-alerts', color: 'good', failOnError: true, message: "Build completed ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)", tokenCredentialId: 'Slack-Integration-Token-SF-CI-ALERTS'
        }
        failure {
            slackSend channel: '#sf-ci-alerts', color: 'danger', failOnError: true, message: "Bad news.  Build failed ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)", tokenCredentialId: 'Slack-Integration-Token-SF-CI-ALERTS'
        }
    }
}

@NonCPS
def jsonParse(def json) {
    new groovy.json.JsonSlurperClassic().parseText(json)
}

@NonCPS
def recordPackageVersionArtifact( def artifactDirectory, def packageVersion ) {
    def fileToFingerprint = "${artifactDirectory}/${packageVersion.Package2Name}-${packageVersion.Package2Id}--v${packageVersion.Version}-branch-${packageVersion.Branch}-${packageVersion.SubscriberPackageVersionId}.packageVersion"
    echo("creating package version artifact for ${fileToFingerprint}")

    writeFile file: fileToFingerprint, text: "${packageVersion}"
}

@NonCPS
def resolvePackageVersion( def allPackageVersionsAvailable, def upstreamDependency ) {

    def result
    def workingPackageVersion
    def workingPackageVersionsAvailableBuildNumber = 0

    for ( packageVersionAvailable in allPackageVersionsAvailable ) {

        if ( upstreamDependency.subscriberPackageVersionId != null && upstreamDependency.subscriberPackageVersionId == packageVersionAvailable.SubscriberPackageVersionId) {
            result = packageVersionAvailable
            break
        }

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