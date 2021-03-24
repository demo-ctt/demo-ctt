pipeline {

    triggers { 
        //TRIGGER DEV (30 em 30 minutos)
        //TRIGGER SIT (1x/dia) as (10 da noite) de (segunda a sexta)
	    parameterizedCron('''
		    */30 * * * 1-5 %Ambiente=DEV
		    * 22 * * 1-5 %Ambiente=SIT
		    ''') }         
    
    agent { label 'master' }
    
    tools { maven "maven 3.6.3" }
    
    options{
        //MANTEM MAXIMO 10 ARTEFACTOS ARQUIVADOS
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '10'))      
    }
    parameters{
        //OPCAO BUILD WITH PARAMETERS
	string(name: 'Ambiente', defaultValue: 'DEV')
	string(name: 'Ambiente', defaultValue: 'SIT')
        choice(choices: 'DEV\nSIT\nquality', description: '', name: 'Ambiente')  
    }

    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "https"
        NEXUS_URL = "devops-nexus.internal.bancopostal.pt"
        NEXUS_REPOSITORY = "maven-nexus-repo/"
        NEXUS_CREDENTIAL_ID = "NEXUS_CREDENTIALS" 
        //VAR DE CONTROLO
        GLOBAL_ENVIRONMENT = "NO BRANCH"    	
        //STRING DO SISTEMA EM CASO DE TRIGGER POR TIMER 
        TIMER = "Started by timer"          
        //STRING DO SISTEMA EM CASO DE TRIGGER POR ADMIN
        ADMIN = "Started by user"
        TEAMS_URL = "MS_TEAMS_BUILDS_DEPLOYS_WEBHOOK_URL"


        //Microsoft Teams Notification Colors (Hexadecimal Color)
        msg_color_success = "#00CC00"
        msg_color_failure = "#FF0000"
        msg_color_aborted = "#A6A6A6"
    }

    
    stages {  
        stage("Setup"){
            steps{
                script{ 
                    def admincause = currentBuild.getBuildCauses()[0].shortDescription.contains(ADMIN)
                    def timercause = currentBuild.getBuildCauses()[0].shortDescription.contains(TIMER)
                    //CASO NAO SEJA POR TIMER OU ADMIN
                    if(!(timercause || admincause)){        
                        //VAR ORIGINADA DO PULL REQUEST. DETERMINA O AMBIENTE(DEV, SIT, QUA, PROD)
                        switch (env.ghprbTargetBranch){ 
                            //BRANCH develop      
                            case 'develop':
                                GLOBAL_ENVIRONMENT = 'develop'
                                break
                            //BRANCH quality
                            case 'quality':
                                GLOBAL_ENVIRONMENT = 'quality'
                                break
                            //QUALQUER BRANCH HOTFIX
                            case 'hotfix/*':
                                GLOBAL_ENVIRONMENT = 'hotfix'
                                break
                            default:
                                GLOBAL_ENVIRONMENT = "NO BRANCH"
                                break
                        }
                    //CASO SEJA TIMER
                    }else if(timercause){           
                        //CASO SELECIONE DEV
                        if("${params.Ambiente}" == "DEV"){        
                            GLOBAL_ENVIRONMENT = 'develop' 
                            sh "git checkout develop"
			            }
                        //CASO SELECIONE SIT       
			            if("${params.Ambiente}" == "SIT"){        
                            GLOBAL_ENVIRONMENT = 'SIT' 
                            sh "git checkout develop" 
			            }	
                    }else if(admincause){           
                        //CASO SELECIONE DEV
                        if("${params.Ambiente}" == "DEV"){        
                            GLOBAL_ENVIRONMENT = 'develop' 
                            sh "git checkout develop"
			            }
            //CASO SELECIONE SIT       
			        else if("${params.Ambiente}" == "SIT"){        
                            GLOBAL_ENVIRONMENT = 'SIT' 
                            sh "git checkout develop"
                            echo "GOES TO SIT"
                        //CASO SELECIONE quality 
                        }else if("${params.Ambiente}" == "quality"){                                  
                            GLOBAL_ENVIRONMENT = 'quality'
                            sh "git checkout quality"
                            echo "GOES TO quality"
                        }
                    }
                    
                }
            }
        }
	    
	stage('READ POM'){
            steps{
                script{
                    //READ POM
                    pom = readMavenPom file: "pom.xml" 
                    //GET POM VERSION
                    version = "${pom.version}"
		    echo "${version}"   
                }
            }
        }

       stage("SIT") {
            steps {
		echo "${version}"
                script {                                                                                                     
                    if(GLOBAL_ENVIRONMENT == 'SIT'){         
                        //CASO CONTENHA -SNAPSHOT
                        if((version.contains("-SNAPSHOT"))){                                                                           
                            //ADICIONA Á VERSAO.(DATA DA BUILD)
                            sh "mvn -q versions:set -DnewVersion=${pom.version}-${BUILD_TIMESTAMP}"    
                            //mvn build-helper:parse-version versions:set -DnewVersion=\${parsedVersion.majorVersion}.\${parsedVersion.minorVersion}.\${parsedVersion.nextIncrementalVersion} versions:commit             
                        }else{ 
                            //ADICIONA Á VERSAO.(-SNAPSHOT).(DATA DA BUILD)  	                                                                           
                            sh "mvn -q versions:set -DnewVersion=${pom.version}-SNAPSHOT-$BUILD_TIMESTAMP"               
                        }
                        //PACKAGE(MAVEN)
                        sh "mvn package -DskipTests=true"                                                                                                     
                    }   
                }
            }
       }  
	    
        stage("DEV") {       
            steps {
		echo "${version}"
                script {                                                                                              	
                    if(GLOBAL_ENVIRONMENT == 'develop'){   
                        //CASO NAO CONTENHA (-SNAPSHOT)                             
                        if(!(version.contains("-SNAPSHOT"))){
                            //VERSAO ADICIONA SNAPSHOT (MAVEN)                                                                                            
                            sh "mvn -q versions:set -DnewVersion=${pom.version}-SNAPSHOT"              
                        } 
                        //PACKAGE(MAVEN)
                        sh "mvn package -DskipTests=true"      
                    }
            	}
            }
        }
        
         stage("quality") {       
            steps {
		echo "${version}"
                script {
                    if(GLOBAL_ENVIRONMENT == 'quality'){      
                        //CASO CONTENHA (-SNAPSHOT)              
                        if((version.contains("-SNAPSHOT"))){     
                            echo "BUILD"
                            //REMOVE -SNAPSHOT
                            sh 'mvn versions:set -DremoveSnapshot'
                            //sh 'mvn versions:commit'
                        }
                        if((version.contains("-HOTFIX"))){
                            //COLOCA APENAS A VERSAO
                            sh 'mvn build-helper:parse-version versions:set -DnewVersion=\'${parsedVersion.majorVersion}.\${parsedVersion.minorVersion}.\${parsedVersion.patchVersion}\' versions:commit'
                        }
                        sh "mvn package -DskipTests=true"   
                    }   
                }
            }
        }


         stage("HOTFIX") {       
            steps {
		echo "${version}"
                script {
                    if(GLOBAL_ENVIRONMENT == 'hotfix'){      
                        //CASO CONTENHA (-HOTFIX)            
                        if((version.contains("-HOTFIX"))){     
                            //INCREMENTO DE PATCH VERSION
                            sh 'mvn build-helper:parse-version versions:set -DnewVersion=\'${parsedVersion.majorVersion}.\${parsedVersion.minorVersion}.\${parsedVersion.nextPatchVersion}-HOTFIX\' versions:commit'
                        }
                        sh "mvn package -DskipTests=true"   
                    }   
                }
            }
        }

        stage("Nexus Repository Artifact") {
            steps { 
		echo "${pom}"
                script {
                    if(GLOBAL_ENVIRONMENT != "NO BRANCH"){
                        echo "INSIDE NEXUS PUBLISHER"
			def pom = readMavenPom file: "pom.xml";
                        //NOME DO ARTEFACTO     
                        def artifactName = "${pom.artifactId}-${pom.version}-${pom.packaging}.jar" 
                        //CAMINHO DO ARTEFACTO
                        def artifactPath = "target/${artifactName}" 

                        //FAZ UPLOAD PARA NEXUS
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION, protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL, groupId: pom.groupId,
                            version: pom.version, repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,

                            artifacts: [
                                [artifactId: pom.artifactId, classifier: '', file: artifactPath, type: pom.packaging],
                                [artifactId: pom.artifactId, classifier: '', file: "pom.xml", type: "pom"]
                            ]
                        );
                    }
                }
            }
        }
    }
    post{ 
        always{  
            deleteDir() 
        }
        success{ 
            echo "Success"      

            script{
                //Pipeline status specific message
                msg_build_info = " > __Success__ after ${currentBuild.durationString.replace(' and counting', '')}"
                
                //Report the status to Teams
                office365ConnectorSend message: "${msg_build_info}", status: "Sucess" ,webhookUrl: TEAMS_URL, color: msg_color_success
            }
        }
        failure{
            echo "Failure"  

            script{
                //Pipeline status specific message
                msg_build_info = " > __Failure__ after ${currentBuild.durationString.replace(' and counting', '')}"

                //Report the status to Teams
                office365ConnectorSend message: "${msg_build_info}", status: "Failure" ,webhookUrl: TEAMS_URL, color: msg_color_failure
            }
        }
        aborted{
            echo "Aborted"   

            script{
                //Pipeline status specific message
                msg_build_info = " > __Aborted__ after ${currentBuild.durationString.replace(' and counting', '')}"

                //Report the status to Teams
                office365ConnectorSend message: "${msg_build_info}", status: "Aborted" ,webhookUrl: TEAMS_URL, color: msg_color_aborted

            }
        }
    }
}
