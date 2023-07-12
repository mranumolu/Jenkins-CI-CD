#!/usr/bin/env groovy

@Library(['com.optum.jenkins.pipeline.library@master']) globalPipelines

pipeline {
    agent {
        label 'docker-maven-slave'
    }
    tools {
        maven 'Maven'
    }
    environment {
        //Application Name
        API_NAME = 'smf-consent-api'

        //GitHub Credentials
        GIT_CREDENTIALS_ID = 'spec_pat_svc_token'

        //ACR Login servers
        ACR_LOGIN_SERVER = 'sppatienttstuscacr.azurecr.io'

        //AKS Service Name
        SERVICE_NAME = "${env.API_NAME}-svc"

        //Docker Images
        TEST_IMAGE = "${env.API_NAME}-test"
        TEST_1_IMAGE = "${env.API_NAME}-test-1"
        STAGE_IMAGE = "${env.API_NAME}"

        //AKS Resource Groups
        TEST_AKS_RESOURCE_GROUP = 'sp-patient-tst-us-c-rg'
        STAGE_AKS_RESOURCE_GROUP = 'sp-patient-stg-us-c-rg'

        //AKS Clusters
        TEST_AKS_CLUSTER = 'sp-patient-tst-us-c-aks'
        STAGE_AKS_CLUSTER = 'sp-patient-stg-us-c-aks'

        //AKS NAmespaces
        TEST_AUTH = 'auth-test'
        TEST_UNAUTH = 'unauth-test'
        TEST_1_AUTH = 'auth-test-1'
        TEST_1_UNAUTH = 'unauth-test-1'
        STAGE_AUTH = 'auth-stage'
        STAGE_UNAUTH = 'unauth-stage'

        //GitHub Branch
        GITHUB_BRANCH = "${GIT_BRANCH.split("/")[1]}"

    }
    stages {
        stage('Maven Build') {
            steps {
                glMavenBuild gitUserCredentialsId: "${env.GIT_CREDENTIALS_ID}",
                mavenGoals: 'clean install'
            }
        }
        stage('Sonar Scan') {
                steps {
                    catchError(buildResult: 'SUCCESS', stageResult:'FAILURE')
                    {
                    glSonarMavenScan gitUserCredentialsId: "${env.GIT_CREDENTIALS_ID}",
                    mainBranchName: 'main',
                    sonarExclusions: '**/domain/*,**/mock/*',
                    additionalProps: ['sonar.sources': 'src/',
                                      'sonar.coverage.jacoco.xmlReportPaths': 'target/jacoco/jacoco.xml'
                                     ]
                }
            }
        }
        stage('Build docker image ') {
            parallel {
                stage('Test') {
                    when { environment name: 'ENVIRONMENT_NAME', value: 'test' }
                    steps {
                        sh """
                    cp $WORKSPACE/target/*.jar $WORKSPACE/deployment/
                    cd $WORKSPACE/deployment/
                    docker build --force-rm --no-cache -t $ACR_LOGIN_SERVER/$TEST_IMAGE:v$BUILD_NUMBER -f $WORKSPACE/deployment/Test.Dockerfile .
                    """
                    }
                }

                    stage('Test-1') {
                    when { environment name: 'ENVIRONMENT_NAME', value: 'test-1' }
                    steps {
                        sh """
                    cp $WORKSPACE/target/*.jar $WORKSPACE/deployment/
                    cd $WORKSPACE/deployment/
                    docker build --force-rm --no-cache -t $ACR_LOGIN_SERVER/$TEST_1_IMAGE:v$BUILD_NUMBER -f $WORKSPACE/deployment/Test.Dockerfile .
                    """
                    }
                    }
                    stage('Stage') {
                    when { environment name: 'ENVIRONMENT_NAME', value: 'stage' }
                    steps {
                        sh """
                    cp $WORKSPACE/target/*.jar $WORKSPACE/deployment/
                    cd $WORKSPACE/deployment/
                    docker build --force-rm --no-cache -t $ACR_LOGIN_SERVER/$STAGE_IMAGE:${GITHUB_BRANCH}_version_${BUILD_NUMBER} -f $WORKSPACE/deployment/Dockerfile .
                    """
                    }
                    }
            }
        }
        stage('Twistlock Scan') {
            parallel {
                stage('Test') {
                    when { environment name: 'ENVIRONMENT_NAME', value: 'test' }
                    steps {
                        glTwistlockScan dockerRepository: "$ACR_LOGIN_SERVER/$TEST_IMAGE:v$BUILD_NUMBER",
                        twistlockCredentials: "Saral's-Docker-credentials",
                        failBuild: false
                    }
                }
                    stage('Test-1') {
                    when { environment name: 'ENVIRONMENT_NAME', value: 'test-1' }
                    steps {
                        glTwistlockScan dockerRepository: "$ACR_LOGIN_SERVER/$TEST_1_IMAGE:v$BUILD_NUMBER",
                        twistlockCredentials: "Saral's-Docker-credentials",
                        failBuild: false
                    }
                    }
                    stage('Stage') {
                    when { environment name: 'ENVIRONMENT_NAME', value: 'stage' }
                    steps {
                        glTwistlockScan dockerRepository: "$ACR_LOGIN_SERVER/$STAGE_IMAGE:${GITHUB_BRANCH}_version_${BUILD_NUMBER}",
                        twistlockCredentials: "Saral's-Docker-credentials",
                        failBuild: false
                    }
                    }
            }
        }
        stage('Push docker image to ACR') {
            parallel {
                stage('Test') {
                    when { environment name: 'ENVIRONMENT_NAME', value: 'test' }
                    steps {
                        glDockerImagePush image: "$ACR_LOGIN_SERVER/$TEST_IMAGE:v$BUILD_NUMBER",
                        credentialsId:'patient_nonprod_acr_credentials'
                    }
                }
                    stage('Test-1') {
                    when { environment name: 'ENVIRONMENT_NAME', value: 'test-1' }
                    steps {
                        glDockerImagePush image: "$ACR_LOGIN_SERVER/$TEST_1_IMAGE:v$BUILD_NUMBER",
                        credentialsId:'patient_nonprod_acr_credentials'
                    }
                    }
                stage('Stage') {
                    when { environment name: 'ENVIRONMENT_NAME', value: 'stage' }
                    steps {
                        glDockerImagePush image: "$ACR_LOGIN_SERVER/$STAGE_IMAGE:${GITHUB_BRANCH}_version_${BUILD_NUMBER}",
                        credentialsId:'patient_nonprod_acr_credentials'
                    }
                }
            }
        }
        stage('Deploy to k8s cluster') {
            parallel {
                stage('Test') {
                    when { environment name: 'ENVIRONMENT_NAME', value: 'test' }
                    steps {
                        glAzureLogin('skyline-specialty-portals-infra-sp-nonprod') {
                            command """
                        az aks get-credentials --resource-group $TEST_AKS_RESOURCE_GROUP --name $TEST_AKS_CLUSTER

                        echo "Create or update the deployment"
                        sed -i "s/<acr-name>/$ACR_LOGIN_SERVER/" $WORKSPACE/deployment/deploy.yml
                        sed -i "s/<image-name>/$TEST_IMAGE/" $WORKSPACE/deployment/deploy.yml
                        sed -i "s/<tag>/v$BUILD_NUMBER/" $WORKSPACE/deployment/deploy.yml
                        cat $WORKSPACE/deployment/deploy.yml
                        kubectl --namespace=$TEST_AUTH apply -f $WORKSPACE/deployment/deploy.yml

                        echo "Create or update the service"
                        kubectl --namespace=${env.TEST_AUTH} get services > services_list.txt
                        if grep "$SERVICE_NAME" services_list.txt
                        then echo "Service $SERVICE_NAME already exists. No need to create again."
                        else echo "Service $SERVICE_NAME is NOT deployed. Deploying now."
                        cat $WORKSPACE/deployment/service.yml
                        kubectl --namespace=$TEST_AUTH create -f $WORKSPACE/deployment/service.yml
                        fi
                        """
                        }
                    }
                }
                stage('Test-1') {
                    when { environment name: 'ENVIRONMENT_NAME', value: 'test-1' }
                    steps {
                        glAzureLogin('skyline-specialty-portals-infra-sp-nonprod') {
                            command """
                        az aks get-credentials --resource-group $TEST_AKS_RESOURCE_GROUP --name $TEST_AKS_CLUSTER

                        echo "Create or update the deployment"
                        sed -i "s/<acr-name>/$ACR_LOGIN_SERVER/" $WORKSPACE/deployment/deploy.yml
                        sed -i "s/<image-name>/$TEST_1_IMAGE/" $WORKSPACE/deployment/deploy.yml
                        sed -i "s/<tag>/v$BUILD_NUMBER/" $WORKSPACE/deployment/deploy.yml
                        cat $WORKSPACE/deployment/deploy.yml
                        kubectl --namespace=$TEST_1_AUTH apply -f $WORKSPACE/deployment/deploy.yml

                        echo "Create or update the service"
                        kubectl --namespace=${env.TEST_1_AUTH} get services > services_list.txt
                        if grep "$SERVICE_NAME" services_list.txt
                        then echo "Service $SERVICE_NAME already exists. No need to create again."
                        else echo "Service $SERVICE_NAME is NOT deployed. Deploying now."
                        cat $WORKSPACE/deployment/service.yml
                        kubectl --namespace=$TEST_1_AUTH create -f $WORKSPACE/deployment/service.yml
                        fi
                        """
                        }
                    }
                }
                stage('Stage') {
                    when { environment name: 'ENVIRONMENT_NAME', value: 'stage' }
                    steps {
                        glAzureLogin('skyline-specialty-portals-infra-sp-nonprod') {
                            command """
                        az aks get-credentials --resource-group $STAGE_AKS_RESOURCE_GROUP --name $STAGE_AKS_CLUSTER

                        echo "Create or update the deployment"
                        sed -i "s/<acr-name>/$ACR_LOGIN_SERVER/" $WORKSPACE/deployment/deploy.yml
                        sed -i "s/<image-name>/$STAGE_IMAGE/" $WORKSPACE/deployment/deploy.yml
                        sed -i "s/<tag>/${GITHUB_BRANCH}_version_${BUILD_NUMBER}/" $WORKSPACE/deployment/deploy.yml
                        cat $WORKSPACE/deployment/deploy.yml
                        kubectl --namespace=$STAGE_AUTH apply -f $WORKSPACE/deployment/deploy.yml

                        echo "Create or update the service"
                        kubectl --namespace=${env.STAGE_AUTH} get services > services_list.txt
                        if grep "$SERVICE_NAME" services_list.txt
                        then echo "Service $SERVICE_NAME already exists. No need to create again."
                        else echo "Service $SERVICE_NAME is NOT deployed. Deploying now."
                        cat $WORKSPACE/deployment/service.yml
                        kubectl --namespace=$STAGE_AUTH create -f $WORKSPACE/deployment/service.yml
                        fi
                        """
                        }
                    }
                }
            }
        }
    }
}
