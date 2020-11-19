pipeline {
    triggers { cron('0 22 * * 1-5') }       //TRIGGER SIT (1x/dia) as (10 da noite) de (segunda a sexta)  
    
    agent { label 'master' }
    
    tools { maven "Maven" }
    
    options{
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '10'))      //MANTEM MAXIMO 10 ARTEFACTOS ARQUIVADOS
        disableConcurrentBuilds()                                                       //SEM BUILDS SIMULTANEAS
    }

    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "172.17.0.1:8081"
        NEXUS_REPOSITORY = "maven-nexus-repo"
        NEXUS_CREDENTIAL_ID = "nexus-credentials" 
        GLOBAL_ENVIRONMENT = "NO BRANCH"    //VAR DE CONTROLO	 
        TIMER = "Started by timer"          //STRING DO SISTEMA EM CASO DE TRIGGER POR TIMER
    }
  
    stages {  
        stage("Setup env"){
            steps{
                script{
                    def cause=currentBuild.getBuildCauses()[0].shortDescription   //VERIFICA SE A CAUSA DA BUILD FOI DE TIMER
                    if(!(cause.contains(TIMER))){
                        switch (env.ghprbTargetBranch){     //VAR ORIGINADA DO PULL REQUEST. DETERMINA O AMBIENTE(DEV, SIT, QUA, PROD)  
                            case 'develop':
                                GLOBAL_ENVIRONMENT = 'develop'
                                break
                            case 'qualidade':
                                GLOBAL_ENVIRONMENT = 'qualidade'
                                break
                            case 'master':
                                GLOBAL_ENVIRONMENT = 'producao'
                                break
                            default:
                                GLOBAL_ENVIRONMENT = "NO BRANCH"
                                break
                        }
                    }else{
                        GLOBAL_ENVIRONMENT = 'SIT'
                    }
                }
            }
        }

       stage("SIT Artifact") {
            steps {
                script {                                                                                                     
                    if(GLOBAL_ENVIRONMENT == 'SIT'){
                        def pom = readMavenPom file: "pom.xml"  //LE POM
                        def version = "${pom.version}"          //APENAS A VERSAO(Ex:1.2)
                    
                        if((version.contains("-SNAPSHOT"))){      //CASO CONTENHA -SNAPSHOT                                                                     
                            sh "mvn -q versions:set -DnewVersion=${pom.version}-$BUILD_TIMESTAMP"  //ADICIONA Á VERSAO.(DATA DA BUILD)
                        }else{   	                                                                           
                            sh "mvn -q versions:set -DnewVersion=${pom.version}-SNAPSHOT-$BUILD_TIMESTAMP"  //ADICIONA Á VERSAO.(-SNAPSHOT).(DATA DA BUILD)                   
                        }
                    sh "mvn package -DskipTests=true"       //PACKAGE(MAVEN)                                                                                            
                    }   
                }
            }
       }  
	    
        stage("DEV Artifact") {       
            steps {
                script {                                                                                              	
                    if(GLOBAL_ENVIRONMENT == 'develop'){     
                        def pom = readMavenPom file: "pom.xml"     //LE
                        def version = "${pom.version}"             //APENAS A VERSAO(Ex:1.2)                 
                        if(!(version.contains("-SNAPSHOT"))){           //CASO CONTENHA (-SNAPSHOT)                                                                                 
                            sh "mvn -q versions:set -DnewVersion=${pom.version}-SNAPSHOT"    //VERSAO COM SNAPSHOT (MAVEN)            
                        } 
                    sh "mvn package -DskipTests=true"   //PACKAGE(MAVEN) 
                    }
            	}
            }
        }
        
         stage("Qualidade Artifact") {       
            steps {
                script {
                    if(GLOBAL_ENVIRONMENT == 'qualidade'){      //QUALIDADE
                        def pom = readMavenPom file: "pom.xml"  //LE POM
                        def version = "${pom.version}"          //APENAS A VERSAO(Ex:1.2)     
                        if((version.contains("-SNAPSHOT-${BUILD_TIMESTAMP}"))){     //CASO CONTENHA (-SNAPSHOT).(DATA DA BUILD)
                            //OU USAR MVN RELEASE???????
                            //INCREMENTAR VERSAO
                            sh "mvn -q versions:set -DnewVersion=${pom.version}"    //VERSAO SEM SNAPSHOT(MAVEN)
                        }else if((version.contains("-SNAPSHOT"))){
                            sh "mvn -q versions:set -DnewVersion=${pom.version}"    //VERSAO SEM SNAPSHOT(MAVEN)  
                            sh "mvn -q versions:use-next-versions"   
                        }
                    sh "mvn -q versions:use-next-versions" 
                   // sh "mvn package -DskipTests=true"   //PACKAGE(MAVEN)
                    }
            	}
            }
        }
        
        stage("Nexus Repository") {
            steps { 
                script {
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