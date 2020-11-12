pipeline {
    triggers { cron('*/5 * * * 1-5') }       
    
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
    }


    stages {  
        stage("Build DEV") {
            steps{
            	script{
                    def causes = currentBuild.getBuildCauses()
                    echo "${causes}"
            //	when{ expression { ghprbTargetBranch == 'develop' } }
		        sh 'printenv'

                    def pom = readMavenPom file: "pom.xml"
                    def version = "${pom.version}"


                    
                    if(!(version.contains("-SNAPSHOT"))){
                        sh "mvn -q versions:set -DnewVersion=${pom.version}-SNAPSHOT" 
                        echo "contem snapshot"
                    } 
                    sh "mvn package -DskipTests=true"
                    echo "build dev com sucesso"
                   }
                }
            }
        }
        
     
        stage("Build SIT") {
            steps{
            	script{
            	if((manager.logContains("Started by timer"))){
                    def pom = readMavenPom file: "pom.xml"
                    def version = "${pom.version}"
                    
                    if((version.contains("-SNAPSHOT"))){
                        sh "mvn -q versions:set -DnewVersion=${pom.version}-$BUILD_TIMESTAMP"
                    }else{
                        sh "mvn -q versions:set -DnewVersion=${pom.version}-SNAPSHOT-$BUILD_TIMESTAMP"
                    }
                    sh "mvn package -DskipTests=true"
                }  
                }          
            }
         }
        
       
        
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

