pipeline {
    agent any
    tools { 
        maven 'Maven-360' 
        jdk 'JAVA-11' 
    }
    /*environment {
        PATH = "$PATH:/opt/apache-maven-3.8.2/bin"
    }*/

    stages {
        stage('Clone the Code') {
            steps {
                sh 'printenv'
                git branch: '$BRANCH_NAME', url: 'https://github.com/futuretechdevops/hello-world.git'
                sh 'echo --------'
                sh 'git branch'
                //git  'https://github.com/futuretechdevops/hello-world.git'
            }
        }
        stage('MVN Build and Publish the Unit Test Results') {
            steps {
                sh 'mvn clean install'
            }
            post {
                always {
                    junit(
                        allowEmptyResults: true,
                        testResults: '**/target/surefire-reports/*.xml'
                    )
                }
            }
        }
        stage ('Code Quality') {
            environment {
                scannerHome = tool 'sonar-scanner-4.7'
            }
            steps {
                withSonarQubeEnv('SonarQube-Server-CE-9.8') {
                    sh "${scannerHome}/bin/sonar-scanner \
                    -D sonar.projectKey=cicd-demo \
                    -D sonar.exclusions=vendor/**,resources/**,**/*.java"
                }
            }
        }
        stage('SonarQube Quality Gates Check'){
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Artifact-Upload') {
            steps {
                echo "build-version ${env.POM_VERSION}"
                script {
                    env.POM_VERSION = sh(returnStdout: true, script: "cat pom.xml  |grep 'version' |head -1 |awk -F '[><]' '{print \$3}'").trim()
                    env.RELEASE_VERSION = sh(returnStdout: true, script: "cat pom.xml  |grep 'version' |head -1 |awk -F '[><]' '{print \$3}' |awk -F '[-]' '{print \$2}'").trim()
                    sh '[[ -d webapp/target2 ]] && echo "target2 directory exist" || mkdir webapp/target2'
                    sh '[[ $(ls -A webapp/target2) ]] && rm -rf webapp/target2/* || echo "target2 empty directory"'
                    sh 'cp webapp/target/webapp.war webapp/target2/$BUILD_NUMBER-$RELEASE_VERSION-webapp.war' 
                    
                    if (env.RELEASE_VERSION == 'RELEASE') {
                        env.RELEASE_VERSION = "Release"
                        echo "This is RELEASE?"
                        echo "${env.RELEASE_VERSION}"
                    }
                    else {
                        env.RELEASE_VERSION = "Snapshot"
                        echo "This is SNAPSHOT?"
                        echo "${env.RELEASE_VERSION}"
                    }
                    rtServer (
                        id: "jfrog", 
                        url: "http://10.10.0.5:8082",
                        credentialsId: "4fe16cb0-df33-4b63-a117-68c127eacd40"
                    )
                    rtUpload (
                        serverId: 'jfrog',
                        spec: '''{
                            "files": [
                                {
                               "pattern": "webapp/target2/*.war",
                               "target": "artifactory/$RELEASE_VERSION/com/example/maven-project/webapp/$POM_VERSION/"
                                }
                            ]
                        }''',
                        buildName: "$JOB_NAME",
                        buildNumber: "$BUILD_NUMBER"
                    )
                }
                stash includes: '**/target/*.war', name: "WARfile"
                stash includes: 'Dockerfile', name: "Dockerfile"
            }
        }
        stage('docker-build/tag/push') {
            agent {
                label 'docker'
            }
            steps {
                unstash "WARfile"
                unstash "Dockerfile"
                //sh 'printenv'
                //sh 'hostname -i && pwd && ls && whoami'
                sh 'docker build -t futuretechdevops/tomcat:\$BUILD_NUMBER-\$RELEASE_VERSION . -f Dockerfile'
                sh 'docker push futuretechdevops/tomcat:\$BUILD_NUMBER-\$RELEASE_VERSION'
            }
        }
        stage('docker-run') {
            agent {
                label 'docker'
            }
            steps {
                sh 'docker stop tomcat-qa1 || true && docker rm tomcat-qa1 || true'
                sh 'docker run -d --name tomcat-qa1 -p 8090:8090 futuretechdevops/tomcat:\$BUILD_NUMBER-\$RELEASE_VERSION'
            }
        }
    }
}
