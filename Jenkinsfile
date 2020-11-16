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
        /*Workaround para build DEV
        Quando a build vem de timer, o job nao reconhece a var(var default de pull request(GITHUB))*/
        ghprbTargetBranch = "develop"	 
    }
    

    stages {  
       stage("Build SIT") {
            steps {
                script {
                /*verifica causa da build, se for timer executa.*/
                def cause=currentBuild.getBuildCauses()[0].shortDescription
                if(cause.contains('Started by timer')){
                    def pom = readMavenPom file: "pom.xml"
                    def version = "${pom.version}"
                    
                    if((version.contains("-SNAPSHOT"))){
                   	 /*adiciona á versao do (pom) + (data atual da build)*/
                        sh "mvn -q versions:set -DnewVersion=${pom.version}-$BUILD_TIMESTAMP"  
                    }else{
                    	/*adiciona á versao do (pom) + (-snapshot) + (data atual da build).*/
                        sh "mvn -q versions:set -DnewVersion=${pom.version}-SNAPSHOT-$BUILD_TIMESTAMP"
                        
                    }
                    sh "mvn package -DskipTests=true"
                    /*variavel = "SIT" para que DEV execute.*/
                    ghprbTargetBranch = 'SIT'
                }               
                }   
            }
        }  
    	
    
        stage("Build DEV") {       
            steps {
                script {
                /*Verifica se o pull request foi feito para a branch develop
                Var originada do pull request. O valor é redefinido nas variaveis de ambiente.*/ 	
                if(ghprbTargetBranch == 'develop'){
                    def pom = readMavenPom file: "pom.xml"
                    def version = "${pom.version}"
                           
                    if(!(version.contains("-SNAPSHOT"))){
                    /*adiciona a versao do pom com -snapshot.*/
                       sh "mvn -q versions:set -DnewVersion=${pom.version}-SNAPSHOT"                    
                    } 
                    sh "mvn package -DskipTests=true"
                }
            	}
            }
        }
        /*
         stage("Build Qualidade") {       
            steps {
                script {
                //FLAG???
                if(ghprbTargetBranch == 'develop'){
                    def pom = readMavenPom file: "pom.xml"
                    def version = "${pom.version}"
                           
                    if((version.contains("-SNAPSHOT-${BUILD_TIMESTAMP}"))){
                        sh "mvn -q versions:set -DnewVersion=${pom.version}"  
                    }else if((version.contains("-SNAPSHOT"))){
                        sh "mvn -q versions:set -DnewVersion=${pom.version}"           
                    }
                    sh "mvn package -DskipTests=true"
                }
            	}
            }
        }
        */

     
        stage("Publish to Nexus") {
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
