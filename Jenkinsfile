pipeline {
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
    }
    

    stages {  
       stage("Build SIT") {
            steps {
                script {
                def cause=currentBuild.getBuildCauses()[0].shortDescription
                sh 'print ${cause}'
                if(cause.contains('Started by timer')){
                    def pom = readMavenPom file: "pom.xml"
                    def version = "${pom.version}"
                    
                    if((version.contains("-SNAPSHOT"))){
                    	echo "contem snap"
                        sh "mvn -q versions:set -DnewVersion=${pom.version}-$BUILD_TIMESTAMP"  
                    }else{
                    	echo "nao contem snap"
                        sh "mvn -q versions:set -DnewVersion=${pom.version}-SNAPSHOT-$BUILD_TIMESTAMP"
                        
                    }
                    sh "mvn package -DskipTests=true"
                }
                }
                
            }
        }  
    	
    
  
        stage("Build DEV") {       
		when{ 
         	  expression { ghprbTargetBranch == 'develop' }
            	}
            steps {
                script {
                    def pom = readMavenPom file: "pom.xml"
                    def version = "${pom.version}"
                    
                    def cause=currentBuild.getBuildCauses()
			print ${cause}
                           
                           
                    if(!(version.contains("-SNAPSHOT"))){
                       sh "mvn -q versions:set -DnewVersion=${pom.version}-SNAPSHOT"                    
                    } 
                    sh "mvn package -DskipTests=true"
                    echo "build dev com sucesso"
                }
            }
        }
        

     
        stage("Publish to Nexus") {
            steps {
                script {
                    def pom = readMavenPom file: "pom.xml";
                    def artifactName = "${pom.artifactId}.${pom.packaging}"
                    def artifactPath = "target/${artifactName}"

			def cause=currentBuild.getBuildCauses()[0].shortDescription
			print "${cause}"

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
