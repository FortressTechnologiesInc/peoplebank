pipeline {
    agent any
    
    tools {
        jdk 'jdk'
        maven 'maven'
    }
    
    environment {
        SCANNER_HOME = tool 'scanner'
    }
    
    stages {
        stage('1.0 Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        
        stage('2.0 Git Checkout') {
            steps {
                git branch: 'master',  url: 'https://github.com/FortressTechnologiesInc/peoplebank.git'
            }
        }
        
        stage('Spin up MariaDB Container') {
            steps {
                script {
                    // Spin up the MariaDB container
                    sh '''
                    docker run -d --rm --name mariadb \
                    -e MYSQL_ROOT_PASSWORD=root \
                    -p 3306:3306 \
                    --health-cmd='mysqladmin ping --silent' \
                    --health-interval=10s \
                    --health-timeout=5s \
                    --health-retries=5 \
                    mariadb:10.3 --lower_case_table_names=1
                    '''

                    // Wait for the database to be healthy
                    sh '''
                    until [ "`docker inspect -f {{.State.Health.Status}} mariadb`" == "healthy" ]; do
                        echo "Waiting for database to be healthy...";
                        sleep 5;
                    done
                    '''
                }
            }
        }

        stage('Set up JDK 17') {
            steps {
                // Install JDK 17 using Jenkins JDK tool if available, otherwise install manually
                sh 'sudo apt-get install openjdk-17-jdk -y'
                sh 'java -version'
            }
        }

       

        
        stage('5.0 Sonarqube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=PeopleBankOfCanada \
                        -Dsonar.java.binaries=. \
                        -Dsonar.projectKey=PeopleBankOfCanada '''
                }
            }
        }
        
        stage('6.0 Quality Check') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar'
                }
            }
        }
        
        stage('Build Maven Project') {
            steps {
                // Build the Maven project, skipping tests for now
                sh 'mvn clean install -DskipTests=true'
            }
        }
        
        stage('10.0 Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh "docker build -t iscanprint/peoplebank:1.0 ."
                        sh "docker push iscanprint/peoplebank:1.0"
                        sh "docker rmi -f iscanprint/peoplebank:1.0"
                    }
                }
            }
        }
        
        stage('11.0 TRIVY Image Scan') {
            steps {
                sh "trivy image iscanprint/peoplebank:1.0 > trivyimage.txt"
            }
        }
        
        stage('12.0 Clean Workspace') {
            steps {
                cleanWs()
            }
        }
    }
    
    post {
        always {
            emailext attachLog: true,
                subject: "'${currentBuild.result}'",
                body: """
                    <html>
                    <body>
                        <div style="background-color: #FFA07A; padding: 15px; margin-bottom: 15px;">
                            <p style="color: white; font-weight: bold;">Project: ${env.JOB_NAME}</p>
                        </div>
                        <div style="background-color: #90EE90; padding: 15px; margin-bottom: 15px;">
                            <p style="color: white; font-weight: bold;">Build Number: ${env.BUILD_NUMBER}</p>
                        </div>
                        <div style="background-color: #87CEEB; padding: 15px; margin-bottom: 15px;">
                            <p style="color: white; font-weight: bold;">URL: ${env.BUILD_URL}</p>
                        </div>
                    </body>
                    </html>
                """,
                to: 'deniferdavies@gmail.com,mokeleke@gmail.com',
                mimeType: 'text/html'
        }
    }
}
