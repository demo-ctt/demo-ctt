pipeline {
    /*Trigger SIT corre (1x/dia) as (10 da noite) de (segunda a sexta)*/
    triggers { cron('0 22 * * 1-5') }       
    
    agent { label 'master' }
    
    tools { maven "Maven" }
    
    options{
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "172.17.0.1:8081"
        NEXUS_REPOSITORY = "maven-nexus-repo"
        NEXUS_CREDENTIAL_ID = "nexus-credentials" 
        GLOBAL_ENVIRONMENT = "NO BRANCH"	 
        TIMER = "Started by timer"
    }
    
    stages {  
        stage("Setup env"){
            steps{
                script{
                    def cause=currentBuild.getBuildCauses()[0].shortDescription       /*verifica causa da build, se for timer executa.*/
                    if(!(cause.contains(TIMER))){
                        switch (ghprbTargetBranch){     /*Var originada do pull request. Determina se é DEV, SIT, QUA ou PROD */ 
                            case: 'develop'
                                GLOBAL_ENVIRONMENT = 'develop'
                                break
                            case: 'Qualidade'
                                GLOBAL_ENVIRONMENT = 'Qualidade'
                                break
                            case: 'master'
                                GLOBAL_ENVIRONMENT = 'Producao'
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
                        def pom = readMavenPom file: "pom.xml"
                        def version = "${pom.version}"
                    
                        if((version.contains("-SNAPSHOT"))){                                                                           
                            sh "mvn -q versions:set -DnewVersion=${pom.version}-$BUILD_TIMESTAMP"  /*adiciona á versao do (pom) + (data atual da build)*/
                        }else{   	                                                                           
                            sh "mvn -q versions:set -DnewVersion=${pom.version}-SNAPSHOT-$BUILD_TIMESTAMP"  /*adiciona á versao do (pom) + (-snapshot) + (data atual da build).*/                     
                        }
                    sh "mvn package -DskipTests=true"                                                                                        
                    GLOBAL_ENVIRONMENT = 'SIT'  /*variavel = "SIT" para que DEV execute.*/           
                    }   
                }
            }
       }  
    	    
        stage("DEV Artifact") {       
            steps {
                script {                                                                                              	
                    if(GLOBAL_ENVIRONMENT == 'develop'){     
                        def pom = readMavenPom file: "pom.xml"
                        def version = "${pom.version}"
                           
                        if(!(version.contains("-SNAPSHOT"))){                                                                                          
                            sh "mvn -q versions:set -DnewVersion=${pom.version}-SNAPSHOT"         /*adiciona a versao do pom com -snapshot.*/             
                        } 
                    sh "mvn package -DskipTests=true"
                    }
            	}
            }
        }
        
         stage("Qualidade Artifact") {       
            steps {
                script {
                    if(GLOBAL_ENVIRONMENT == 'Qualidade'){
                        def pom = readMavenPom file: "pom.xml"
                        def version = "${pom.version}"
                           
                        if((version.contains("-SNAPSHOT-${BUILD_TIMESTAMP}"))){
                            //OU USAR MVN RELEASE???????
                            //INCREMENTAR VERSAO
                            sh "mvn -q versions:set -DnewVersion=${pom.version}"  
                        }else if((version.contains("-SNAPSHOT"))){
                            sh "mvn -q versions:set -DnewVersion=${pom.version}"           
                        }
                    sh "mvn package -DskipTests=true"
                    }
            	}
            }
        }
        
        stage("Nexus Repository") {
            steps { 
                script {
                    def pom = readMavenPom file: "pom.xml";
                    def artifactName = "${pom.artifactId}.${pom.packaging}"
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