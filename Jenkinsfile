pipeline {
    triggers { cron('0 22 * * 1-5') }       //TRIGGER SIT (1x/dia) as (10 da noite) de (segunda a sexta)  
    
    agent { label 'master' }
    
    tools { maven "Maven" }
    
    options{
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '10'))      //MANTEM MAXIMO 10 ARTEFACTOS ARQUIVADOS
    }
    parameters{
        choice(choices: 'SIT\nqualidade', description: '', name: 'Ambiente')  //OPCOES BUILD WITH PARAMETERS
    }

    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "172.17.0.1:8081"
        NEXUS_REPOSITORY = "maven-nexus-repo"
        NEXUS_CREDENTIAL_ID = "nexus-credentials" 
        GLOBAL_ENVIRONMENT = "NO BRANCH"    //VAR DE CONTROLO	 
        TIMER = "Started by timer"          //STRING DO SISTEMA EM CASO DE TRIGGER POR TIMER
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
                    if(!(timercause || admincause)){        //CASO NAO SEJA POR TIMER OU ADMIN
                        switch (env.ghprbTargetBranch){     //VAR ORIGINADA DO PULL REQUEST. DETERMINA O AMBIENTE(DEV, SIT, QUA, PROD)  
                            case 'develop':
                                GLOBAL_ENVIRONMENT = 'develop'
                                break
                            case 'qualidade':
                                GLOBAL_ENVIRONMENT = 'qualidade'
                                break
                            case 'hotfix/*':
                                GLOBAL_ENVIRONMENT = 'hotfix'
                                break
                            default:
                                GLOBAL_ENVIRONMENT = "NO BRANCH"
                                break
                        }
                    }else if(timercause){           //CASO SEJA TIMER
                        GLOBAL_ENVIRONMENT = 'SIT'
                        sh "git checkout develop"
                        echo "GOES TO SIT"
                    }else if(admincause){           //CASO SEJA ADMIN
                        if("${params.Ambiente}" == "SIT"){        //CASO SELECIONE SIT
                            GLOBAL_ENVIRONMENT = 'SIT' 
                            sh "git checkout develop"
                            echo "GOES TO SIT"                 
                        }else{                                  //CASO SELECIONE QUALIDADE
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
                        def pom = readMavenPom file: "pom.xml"  //LE POM
                        def version = "${pom.version}"          //APENAS A VERSAO(Ex:1.2)
                    
                        if((version.contains("-SNAPSHOT"))){      //CASO CONTENHA -SNAPSHOT                                                                     
                            sh "mvn -q versions:set -DnewVersion=${pom.version}-$BUILD_TIMESTAMP"  //ADICIONA Á VERSAO.(DATA DA BUILD)
                            echo "BUILD VERSION+DATE"
                        }else{   	                                                                           
                            sh "mvn -q versions:set -DnewVersion=${pom.version}-SNAPSHOT-$BUILD_TIMESTAMP"  //ADICIONA Á VERSAO.(-SNAPSHOT).(DATA DA BUILD)    
                            echo "BUILD VERSION+SNAPSHOT+DATE"               
                        }
                        sh "mvn package -DskipTests=true"       //PACKAGE(MAVEN)
                        office365ConnectorSend webhookUrl: TEAMS_URL,
                        message: 'Novo Artefacto SIT disponivel. Version = ${pom.version}',
                        status: 'Success'                                                                                              
                    }   
                }
            }
       }  
	    
        stage("DEV") {       
            steps {
                script {                                                                                              	
                    if(GLOBAL_ENVIRONMENT == 'develop'){ 
                        echo "INSIDE DEV"    
                        def pom = readMavenPom file: "pom.xml"     //LE
                        def version = "${pom.version}"             //APENAS A VERSAO(Ex:1.2)                 
                        if(!(version.contains("-SNAPSHOT"))){           //CASO CONTENHA (-SNAPSHOT)                                                                                 
                            sh "mvn -q versions:set -DnewVersion=${pom.version}-SNAPSHOT"    //VERSAO COM SNAPSHOT (MAVEN) 
                            echo "BUILD VERSION+SNAPSHOT"           
                        } 
                        sh "mvn package -DskipTests=true"   //PACKAGE(MAVEN) 
                        office365ConnectorSend webhookUrl: TEAMS_URL,
                        message: 'Novo Artefacto DEV disponivel.',
                        status: 'Success'  
                    }
            	}
            }
        }
        
         stage("Qualidade") {       
            steps {
                script {
                    if(GLOBAL_ENVIRONMENT == 'qualidade'){      
                        echo "INSIDE QUALIDADE"
                        def pom = readMavenPom file: "pom.xml"  //LE POM
                        def version = "${pom.version}"          //APENAS A VERSAO(Ex:1.2)     
                        if((version.contains("-SNAPSHOT"))){     //CASO CONTENHA (-SNAPSHOT).(DATA DA BUILD)
                            echo "BUILD"
                            sh 'mvn versions:set -DremoveSnapshot'
                            //sh 'mvn validate -Pdrop-snapshot'
                            //sh 'mvn versions:use-releases'
                            sh 'mvn versions:commit'
                            echo "BUILD VERSION ONLY"
                        }
                        if((version.contains("-HOTFIX"))){
                            sh 'mvn build-helper:parse-version versions:set -DnewVersion=\'${parsedVersion.majorVersion}.\${parsedVersion.minorVersion}.\${parsedVersion.patchVersion}\' versions:commit'
                        }
                        sh "mvn package -DskipTests=true" 
                        office365ConnectorSend webhookUrl: TEAMS_URL,
                        message: 'Novo Artefacto QUALIDADE disponivel.',
                        status: 'Success'  
                    }   
                    //INCREMENTO DE MINOR VERSION
                    //sh 'mvn build-helper:parse-version versions:set -DnewVersion=\'${parsedVersion.majorVersion}.\${parsedVersion.nextMinorVersion}\' versions:commit'
                }
            }
        }


         stage("HOTFIX") {       
            steps {
                script {
                    if(GLOBAL_ENVIRONMENT == 'hotfix'){      
                        echo "INSIDE HOTFIX branch"
                        def pom = readMavenPom file: "pom.xml"  //LE POM
                        def version = "${pom.version}"          //APENAS A VERSAO(Ex:1.2)     
                        if((version.contains("-HOTFIX"))){     //CASO CONTENHA (-SNAPSHOT).(DATA DA BUILD)
                            sh 'mvn build-helper:parse-version versions:set -DnewVersion=\'${parsedVersion.majorVersion}.\${parsedVersion.minorVersion}.\${parsedVersion.nextPatchVersion}-HOTFIX\' versions:commit'
                            echo "BUILD HOTFIX"
                        }
                        sh "mvn package -DskipTests=true" 
                        office365ConnectorSend webhookUrl: TEAMS_URL,
                        message: 'Novo Artefacto HOTFIX disponivel.',
                        status: 'Success'  
                    }   
                }
            }
        }

        stage("Nexus Repository Artifact") {
            steps { 
                script {
                    if(GLOBAL_ENVIRONMENT != "NO BRANCH"){
                        echo "INSIDE NEXUS PUBLISHER"
                        def pom = readMavenPom file: "pom.xml";     //LE POM
                        def artifactName = "${pom.artifactId}.${pom.packaging}" //NOME DO ARTEFACTO
                        def artifactPath = "target/${artifactName}" 

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