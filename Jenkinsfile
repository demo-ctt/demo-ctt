pipeline {
    //TRIGGER SIT (1x/dia) as (10 da noite) de (segunda a sexta)
	
    triggers { 
	    parameterizedCron('''
		    */2 * * * * %Ambiente=DEV
		    */5 * * * * %Ambiente=SIT
		    ''') }         
    
    agent { label 'master' }
    
    tools { maven "Maven" }
    
    options{
        //MANTEM MAXIMO 10 ARTEFACTOS ARQUIVADOS
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '10'))      
    }
    parameters{
        //OPCAO BUILD WITH PARAMETERS
	string(name: 'Ambiente', defaultValue: 'DEV')
	string(name: 'Ambiente', defaultValue: 'SIT')
        choice(choices: 'DEV\nSIT\nqualidade', description: '', name: 'Ambiente')  
    }

    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "172.17.0.1:8081"
        NEXUS_REPOSITORY = "maven-nexus-repo"
        NEXUS_CREDENTIAL_ID = "nexus-credentials" 
        //VAR DE CONTROLO
        GLOBAL_ENVIRONMENT = "NO BRANCH"    	
        //STRING DO SISTEMA EM CASO DE TRIGGER POR TIMER 
        TIMER = "Started by timer"          
        //STRING DO SISTEMA EM CASO DE TRIGGER POR ADMIN
        ADMIN = "Started by user"
        TEAMS_URL = "https://outlook.office.com/webhook/27903f47-1649-4be5-8eec-b00ed76b2b6d@39c83d5e-cede-42d1-962f-c6a853ab7cf5/JenkinsCI/15874adb11b140e6bac8331836b4ad29/9469806e-38aa-43c2-b3d6-086656250e72"
    }

    stages {  
        stage("Setup"){
            steps{
                script{ 
                    echo "SETUP ENVIRONMENT"
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
                            //BRANCH qualidade
                            case 'qualidade':
                                GLOBAL_ENVIRONMENT = 'qualidade'
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
				echo "GOES TO develop"
			}
                        //CASO SELECIONE SIT       
			if("${params.Ambiente}" == "SIT"){        
                            GLOBAL_ENVIRONMENT = 'SIT' 
                            sh "git checkout develop"
                            echo "GOES TO SIT"  
			 }	
                    }else if(admincause){           
                        //CASO SELECIONE DEV
                        if("${params.Ambiente}" == "DEV"){        
                            GLOBAL_ENVIRONMENT = 'develop' 
                            sh "git checkout develop"
				echo "GOES TO develop"
			}
                        //CASO SELECIONE SIT       
			if("${params.Ambiente}" == "SIT"){        
                            GLOBAL_ENVIRONMENT = 'SIT' 
                            sh "git checkout develop"
                            echo "GOES TO SIT"
                        //CASO SELECIONE QUALIDADE 
                        }else{                                  
                            GLOBAL_ENVIRONMENT = 'qualidade'
                            sh "git checkout qualidade"
                            echo "GOES TO Qualidade"
                        }
                    }
                    
                }
            }
        }

       stage("SIT") {
            steps {
                script {                                                                                                     
                    if(GLOBAL_ENVIRONMENT == 'SIT'){
                        echo "INSIDE SIT"
                        //LE POM
                        def pom = readMavenPom file: "pom.xml"  
                        //APENAS A VERSAO(Ex:1.2)
                        def version = "${pom.version}"          
                    
                        //CASO CONTENHA -SNAPSHOT
                        if((version.contains("-SNAPSHOT"))){                                                                           
                            //ADICIONA Á VERSAO.(DATA DA BUILD)
                            sh "mvn -q versions:set -DnewVersion=${pom.version}-$BUILD_TIMESTAMP"  
                            echo "BUILD VERSION+DATE"
                            office365ConnectorSend webhookUrl: TEAMS_URL,
                            message: "Novo Artefacto SIT disponivel. Versao -${pom.version}-$BUILD_TIMESTAMP",
                            status: 'Success' 
                        }else{ 
                            //ADICIONA Á VERSAO.(-SNAPSHOT).(DATA DA BUILD)  	                                                                           
                            sh "mvn -q versions:set -DnewVersion=${pom.version}-SNAPSHOT-$BUILD_TIMESTAMP"      
                            echo "BUILD VERSION+SNAPSHOT+DATE"
                            office365ConnectorSend webhookUrl: TEAMS_URL,
                            message: "Novo Artefacto SIT disponivel. Versao -${pom.version}-SNAPSHOT-$BUILD_TIMESTAMP",
                            status: 'Success'               
                        }
                        //PACKAGE(MAVEN)
                        sh "mvn package -DskipTests=true"                                                                                                     
                    }   
                }
            }
       }  
	    
        stage("DEV") {       
            steps {
                script {                                                                                              	
                    if(GLOBAL_ENVIRONMENT == 'develop'){ 
                        echo "INSIDE DEV"    
                        //LE POM
                        def pom = readMavenPom file: "pom.xml" 
                        //APENAS A VERSAO(Ex:1.2)    
                        def version = "${pom.version}" 
                        //CASO NAO CONTENHA (-SNAPSHOT)                             
                        if(!(version.contains("-SNAPSHOT"))){
                            //VERSAO ADICIONA SNAPSHOT (MAVEN)                                                                                            
                            sh "mvn -q versions:set -DnewVersion=${pom.version}-SNAPSHOT"     
                            echo "BUILD VERSION+SNAPSHOT"
                            //NOTIFICA TEAMS
                            office365ConnectorSend webhookUrl: TEAMS_URL,
                            message: "Novo Artefacto DEV disponivel. Versao -${pom.version}-SNAPSHOT",
                            status: 'Success'           
                        } 
                        //PACKAGE(MAVEN)
                        sh "mvn package -DskipTests=true"      
                    }
            	}
            }
        }
        
         stage("Qualidade") {       
            steps {
                script {
                    if(GLOBAL_ENVIRONMENT == 'qualidade'){      
                        echo "INSIDE QUALIDADE"
                        //LE POM
                        def pom = readMavenPom file: "pom.xml" 
                        //APENAS A VERSAO(Ex:1.2) 
                        def version = "${pom.version}" 
                        //CASO CONTENHA (-SNAPSHOT)              
                        if((version.contains("-SNAPSHOT"))){     
                            echo "BUILD"
                            //REMOVE -SNAPSHOT
                            sh 'mvn versions:set -DremoveSnapshot'
                            sh 'mvn versions:commit'
                            echo "BUILD VERSION ONLY"
                            //NOTIFICA TEAMS
                            office365ConnectorSend webhookUrl: TEAMS_URL,
                            message: "Novo Artefacto QUALIDADE disponivel. Versao -${pom.version}",
                            status: 'Success'
                        }
                        if((version.contains("-HOTFIX"))){
                            //COLOCA APENAS A VERSAO
                            sh 'mvn build-helper:parse-version versions:set -DnewVersion=\'${parsedVersion.majorVersion}.\${parsedVersion.minorVersion}.\${parsedVersion.patchVersion}\' versions:commit'
                            //NOTIFICA TEAMS
                            office365ConnectorSend webhookUrl: TEAMS_URL,
                            message: "Novo Artefacto HOTFIX disponivel. Versao -${pom.version}",
                            status: 'Success' 
                        }
                        sh "mvn package -DskipTests=true"   
                    }   
                }
            }
        }


         stage("HOTFIX") {       
            steps {
                script {
                    if(GLOBAL_ENVIRONMENT == 'hotfix'){      
                        echo "INSIDE HOTFIX branch"
                        //LE POM
                        def pom = readMavenPom file: "pom.xml" 
                        //APENAS A VERSAO(Ex:1.2)  
                        def version = "${pom.version}"
                        //CASO CONTENHA (-HOTFIX)            
                        if((version.contains("-HOTFIX"))){     
                            //INCREMENTO DE PATCH VERSION
                            sh 'mvn build-helper:parse-version versions:set -DnewVersion=\'${parsedVersion.majorVersion}.\${parsedVersion.minorVersion}.\${parsedVersion.nextPatchVersion}-HOTFIX\' versions:commit'
                            echo "BUILD HOTFIX"
                            //NOTIFICA TEAMS
                            office365ConnectorSend webhookUrl: TEAMS_URL,
                            message: "Novo Artefacto HOTFIX disponivel. Versao -${parsedVersion.majorVersion}.${parsedVersion.minorVersion}.${parsedVersion.nextPatchVersion}-HOTFIX",
                            status: 'Success'
                        }
                        sh "mvn package -DskipTests=true"   
                    }   
                }
            }
        }

        stage("Nexus Repository Artifact") {
            steps { 
                script {
                    if(GLOBAL_ENVIRONMENT != "NO BRANCH"){
                        echo "INSIDE NEXUS PUBLISHER"
                        //LE POM
                        def pom = readMavenPom file: "pom.xml";
                        //NOME DO ARTEFACTO     
                        def artifactName = "${pom.artifactId}.${pom.packaging}" 
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
}