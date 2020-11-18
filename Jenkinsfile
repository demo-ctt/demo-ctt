pipeline {
    /*Trigger SIT corre (1x/dia) as (10 da noite) de (segunda a sexta)*/
    triggers { cron('0 22 * * 1-5') }       
    
    agent {
            label 'master'
    }
    
    tools { maven "Maven" }
    
    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "172.17.0.1:8081"
        NEXUS_REPOSITORY = "maven-nexus-repo"
        NEXUS_CREDENTIAL_ID = "nexus-credentials"
        /*Workaround para build DEV -Quando a build vem de timer, o job nao reconhece a var(var default de pull request(GITHUB))*/
        ghprbTargetBranch = "develop"
        GLOBAL_ENVIRONMENT = "NO STAGE"	 
        TIMER = "Started by timer"
    }
    
    stages {  
        stage("PIPELINE ENTRY POINT"){
            steps{
                script{
                    switch (env.ghprbTargetBranch){
                        case: 'develop'
                            GLOBAL_ENVIRONMENT = 'develop'
                            break
                        case: 'SIT'
                            GLOBAL_ENVIRONMENT = 'SIT'
                            break
                        case: 'Qualidade'
                            GLOBAL_ENVIRONMENT = 'Qualidade'
                            break
                        case: 'Producao'
                            GLOBAL_ENVIRONMENT = 'Producao'
                            break
                        default:
                            GLOBAL_ENVIRONMENT = "NO STAGE"
                            break
                    }
                }
            }
        }

       stage("SIT") {
            steps {
                script {                                                                                                     
                    def cause=currentBuild.getBuildCauses()[0].shortDescription       /*verifica causa da build, se for timer executa.*/
                    if(cause.contains(TIMER)){
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
    	
    
        stage("DEV") {       
            steps {
                script {                                                                                              	
                    if(GLOBAL_ENVIRONMENT == 'develop'){     /*Var originada do pull request. O valor é redefinido nas variaveis de ambiente.*/ 
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
        
         stage("QUA") {       
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
        
        stage("Publish Nexus") {
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
}
