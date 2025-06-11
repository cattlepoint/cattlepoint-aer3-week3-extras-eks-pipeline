pipeline {
    agent any

    /* ---------- Runtime parameters ---------- */
    parameters {
        string(name: 'CLUSTER_NAME',        defaultValue: 'basic-eks', description: 'EKS cluster / stack name')
        string(name: 'CF_TEMPLATE',         defaultValue: 'eks-deployment.yaml', description: 'CloudFormation template file')
        string(name: 'AWS_REGION',          defaultValue: 'us-east-1',        description: 'Deployment region')
    }

    /* ---------- Global env ---------- */
    environment {
        AWS_DEFAULT_REGION = "${params.AWS_REGION}"
        STACK_NAME        = "${params.CLUSTER_NAME}"
        TEMPLATE_FILE     = "${params.CF_TEMPLATE}"
    }

    stages {

        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Validate template') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'eks-deploy']]) {
                    sh "aws cloudformation validate-template --template-body file://${TEMPLATE_FILE}"
                }
            }
        }

        stage('Deploy / Update stack') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'eks-deploy']]) {
                    sh '''
                        #!/bin/bash
                        set +e
                        stack_status=$(aws cloudformation describe-stacks --stack-name ${STACK_NAME} --query 'Stacks[0].StackStatus' --output text 2>/dev/null || true)
                        set -e
                        if [ "${stack_status}" = "ROLLBACK_COMPLETE" ]; then
                          echo "Stack ${STACK_NAME} is in ROLLBACK_COMPLETE. Deleting before redeploy..."
                          aws cloudformation delete-stack --stack-name ${STACK_NAME}
                          aws cloudformation wait stack_delete_complete --stack-name ${STACK_NAME}
                        fi
                        aws cloudformation deploy \
                          --stack-name ${STACK_NAME} \
                          --template-file ${TEMPLATE_FILE} \
                          --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM
                    '''
                }
            }
        }

        stage('Show outputs') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'eks-deploy']]) {
                    sh '''
                        aws cloudformation describe-stacks \
                          --stack-name ${STACK_NAME} \
                          --query 'Stacks[0].Outputs' \
                          --output table
                    '''
                }
            }
        }

        stage('Update kubeconfig') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'eks-deploy']]) {
                    sh "aws eks update-kubeconfig --name ${STACK_NAME}"
                }
            }
        }
    }
}
