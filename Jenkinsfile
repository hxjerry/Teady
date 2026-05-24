pipeline {
    agent any

    stages {
        stage('Clean') {
            steps {
                sh 'mvn -B clean'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn -B compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -B test -Dmaven.test.failure.ignore=true'
            }
        }

        stage('Package') {
            steps {
                sh 'mvn -B -U install -DskipTests'
            }
        }

        stage('PMD') {
            steps {
                sh 'mvn -B -U pmd:pmd'
            }
        }

        stage('JaCoCo') {
            steps {
                sh 'mvn -B jacoco:report'
            }
        }

        stage('Javadoc') {
            steps {
                sh 'mvn -B javadoc:javadoc'
            }
        }

        stage('Site') {
            steps {
                sh 'mvn -B -U site -DskipTests'
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '**/target/site/**/*.*', fingerprint: true
            archiveArtifacts artifacts: '**/target/**/*.jar', fingerprint: true
            archiveArtifacts artifacts: '**/target/**/*.war', fingerprint: true
            junit '**/target/surefire-reports/*.xml'
            cleanWs()
        }
    }
}
