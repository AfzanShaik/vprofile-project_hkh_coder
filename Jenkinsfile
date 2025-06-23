pipeline {
    // Defines the agent for the entire pipeline.
    // This will pull the 'maven:3.8.6-jdk-11' Docker image and run all stages inside a container based on this image.
    agent {
        docker {
            image 'maven:3.8.6-jdk-11' // Use a Maven image with JDK 11
            args '-v $HOME/.m2:/root/.m2' // Mount Maven local repository for caching dependencies
        }
    }

    // Removed the 'tools' block as Maven and JDK are now provided by the Docker image.
    // tools {
    //     maven "MAVEN3"
    //     jdk "JDK17"
    // }

    environment {
        // Nexus configuration
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "192.168.0.61:8081" // Ensure this IP is accessible from your Jenkins agent/Docker container
        NEXUS_REPOSITORY = "vprofile-release"
        NEXUS_REPO_ID    = "vprofile-release" // Used for Maven settings.xml if Maven deploy plugin is used
        NEXUS_CREDENTIAL_ID = "admin" // Jenkins credential ID for Nexus authentication

        // Artifact version (using Jenkins BUILD_ID for uniqueness)
        ARTVERSION = "${env.BUILD_ID}"
    }

    stages {
        stage('BUILD') {
            steps {
                // 'mvn clean install -DskipTests' compiles the project and packages it, skipping unit tests
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    // Archive the generated .war artifact
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST') {
            steps {
                // Run unit tests
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST') {
            steps {
                // Run integration tests, skipping unit tests
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage ('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                // Run Checkstyle analysis
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {
            // SonarQube Scanner home is typically configured globally in Jenkins
            // and made available via the 'tool' directive.
            // 'withSonarQubeEnv' activates the SonarQube environment based on the Jenkins configuration.
            environment {
                // Assuming 'sonarscanner4' is a tool configured in Jenkins
                scannerHome = tool 'sonarscanner4'
            }
            steps {
                withSonarQubeEnv('sonar-pro') { // 'sonar-pro' is the SonarQube server configuration ID in Jenkins
                    // Execute SonarQube analysis
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \\
                        -Dsonar.projectName=vprofile-repo \\
                        -Dsonar.projectVersion=1.0 \\
                        -Dsonar.sources=src/ \\
                        -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \\
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \\
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \\
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
                // Wait for the SonarQube Quality Gate to pass within a 10-minute timeout
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Publish to Nexus Repository Manager") {
            steps {
                script {
                    // Read the Maven POM file to get artifact details
                    def pom = readMavenPom file: "pom.xml"
                    // Find the compiled artifact (e.g., .war file) in the target directory
                    def filesByGlob = findFiles(glob: "target/*.${pom.packaging}")

                    // Print artifact details for debugging
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"

                    def artifactPath = filesByGlob[0].path
                    def artifactExists = fileExists artifactPath

                    if (artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version} ARTVERSION"
                        // Upload artifact to Nexus using the nexusArtifactUploader step
                        nexusArtifactUploader(
                            nexusVersion: NEXUS_VERSION,
                            protocol: NEXUS_PROTOCOL,
                            nexusUrl: NEXUS_URL,
                            groupId: pom.groupId,
                            version: ARTVERSION, // Using the custom artifact version
                            repository: NEXUS_REPOSITORY,
                            credentialsId: NEXUS_CREDENTIAL_ID,
                            artifacts: [
                                [artifactId: pom.artifactId,
                                 classifier: '', // No classifier for the main artifact
                                 file: artifactPath,
                                 type: pom.packaging],
                                [artifactId: pom.artifactId,
                                 classifier: '', // No classifier for the POM file
                                 file: "pom.xml",
                                 type: "pom"] // Upload the POM file as well
                            ]
                        )
                    } else {
                        // Fail the pipeline if the artifact is not found
                        error "*** File: ${artifactPath}, could not be found"
                    }
                }
            }
        }
    }
}
