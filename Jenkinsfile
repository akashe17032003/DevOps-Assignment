#!/usr/bin/env groovy

pipeline {

    agent any

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['qa', 'staging', 'prod'],
            description: 'Select environment'
        )

        string(
            name: 'VERSION',
            defaultValue: '',
            description: 'Leave empty to use latest commit'
        )
    }

    environment {
    APP_NAME    = "${env.APP_NAME}"
    GCP_PROJECT = "${env.GCP_PROJECT}"
    GCP_ZONE    = "${env.GCP_ZONE}"
    GCS_BUCKET  = "${env.GCS_BUCKET}"
 
 }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Initialize') {
            steps {
                script {
                    env.APP_VERSION = params.VERSION?.trim()
                        ? params.VERSION
                        : sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()

                    envConfig = getEnvConfig(params.ENVIRONMENT)

                    echo "Environment: ${params.ENVIRONMENT}"
                    echo "Version: ${env.APP_VERSION}"
                }
            }
        }

        stage('Build and Test') {
            steps {
                sh 'mvn clean package -DskipTests=false'
            }
        }

        stage('Authenticate to GCP') {
            steps {
                withCredentials([file(credentialsId: 'GCP_SA_KEY', variable: 'GCP_KEY')]) {
                    sh '''
                        gcloud auth activate-service-account --key-file=$GCP_KEY
                        gcloud config set project $GCP_PROJECT
                    '''
                }
            }
        }

        stage('Upload Artifact') {
            steps {
                sh '''
                    gsutil -m cp target/*.jar $GCS_BUCKET/$APP_VERSION/app.jar
                '''
            }
        }

        stage('Production Approval') {
            when {
                expression { params.ENVIRONMENT == 'prod' }
            }
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    input message: 'Approve Production Deployment?', ok: 'Deploy'
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    deploy(envConfig, env.APP_VERSION)
                }
            }
        }
    }

    post {
        success {
            echo "Deployment successful"
        }
        failure {
            echo "Deployment failed"
        }
        always {
            cleanWs()
        }
    }
}

def getEnvConfig(String env) {

    def config = [
        qa: [
            vms: ['sync-service-qa-vm'],
            profile: 'qa',
            secret: 'MONGO_URI_QA'
        ],
        staging: [
            vms: ['sync-service-staging-vm'],
            profile: 'staging',
            secret: 'MONGO_URI_STAGING'
        ],
        prod: [
            vms: ['sync-service-prod-01', 'sync-service-prod-02'],
            profile: 'prod',
            secret: 'MONGO_URI_PROD'
        ]
    ]

    if (!config.containsKey(env)) {
        error "Invalid environment: ${env}"
    }

    return config[env]
}

def deploy(Map config, String version) {

    def vms = config.vms
    def profile = config.profile
    def secretName = config.secret

    def mongoUri = sh(
        script: "gcloud secrets versions access latest --secret=${secretName}",
        returnStdout: true
    ).trim()

    def encodedUri = mongoUri.bytes.encodeBase64().toString()

    for (vm in vms) {

        echo "Deploying ${version} to ${vm}"

        def cmd = """
        sudo /opt/scripts/deploy.sh ${version} ${profile} \
        --mongo-uri-base64 '${encodedUri}'
        """

        sh """
            gcloud compute ssh ${vm} \
            --zone=${env.GCP_ZONE} \
            --project=${env.GCP_PROJECT} \
            --tunnel-through-iap \
            --command="${cmd}"
        """

        sleep 10

        def retries = 5
        def health = ''

        for (int i = 0; i < retries; i++) {

            health = sh(
                script: """
                    gcloud compute ssh ${vm} \
                    --zone=${env.GCP_ZONE} \
                    --project=${env.GCP_PROJECT} \
                    --tunnel-through-iap \
                    --command="curl -s -o /dev/null -w '%{http_code}' http://localhost:8080/actuator/health"
                """,
                returnStdout: true
            ).trim()

            if (health == '200') {
                echo "${vm} is healthy"
                break
            }

            echo "Waiting for service on ${vm}"
            sleep 10
        }

        if (health != '200') {
            error "Health check failed on ${vm}"
        }
    }
}