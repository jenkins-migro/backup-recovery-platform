<![CDATA[
// Source: https://github.com/jenkinsci/pipeline-examples (MIT License)
// Test case: Android build with flavors based on branch name
pipeline {
    agent { label 'android' }
    
    environment {
        ANDROID_HOME = '/opt/android-sdk'
        GRADLE_OPTS = '-Dorg.gradle.daemon=false'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Setup') {
            steps {
                sh '''
                    echo "Android SDK: $ANDROID_HOME"
                    echo "Branch: $BRANCH_NAME"
                    
                    # Extract flavor from branch name (QA_flavorname)
                    if [[ "$BRANCH_NAME" =~ ^QA_(.+)$ ]]; then
                        echo "FLAVOR=${BASH_REMATCH[1]}" > flavor.properties
                        echo "Building flavor: ${BASH_REMATCH[1]}"
                    else
                        echo "FLAVOR=debug" > flavor.properties
                        echo "Default flavor: debug"
                    fi
                '''
                
                script {
                    def props = readProperties file: 'flavor.properties'
                    env.BUILD_FLAVOR = props.FLAVOR
                }
            }
        }
        
        stage('Build') {
            steps {
                sh """
                    ./gradlew clean
                    ./gradlew assemble${env.BUILD_FLAVOR.capitalize()}
                """
            }
        }
        
        stage('Test') {
            steps {
                sh "./gradlew test${env.BUILD_FLAVOR.capitalize()}UnitTest"
                publishTestResults testResultsPattern: '**/TEST-*.xml'
            }
        }
        
        stage('Lint') {
            steps {
                sh "./gradlew lint${env.BUILD_FLAVOR.capitalize()}"
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'app/build/reports/lint-results-' + env.BUILD_FLAVOR,
                    reportFiles: 'lint-results.html',
                    reportName: 'Lint Report'
                ])
            }
        }
        
        stage('Archive') {
            steps {
                archiveArtifacts artifacts: "app/build/outputs/apk/${env.BUILD_FLAVOR}/*.apk", fingerprint: true
            }
        }
        
        stage('Deploy') {
            when {
                expression { env.BUILD_FLAVOR != 'debug' }
            }
            steps {
                echo "Deploying ${env.BUILD_FLAVOR} APK to test distribution"
                // Add deployment steps here
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}
    ]]>