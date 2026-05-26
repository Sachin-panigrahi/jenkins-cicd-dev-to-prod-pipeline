pipeline {
    environment {
        JENKINS_URL = credentials('JENKINS_URL')
        JENKINS_USER = credentials('JENKINS_USER')
        JENKINS_API_TOKEN = credentials('JENKINS_API_TOKEN')
        SSH_CREDENTIALS = 'ssh-to-DTVM'
        REMOTE_HOST = 'ubuntu@host' // example: ubuntu@78.87.99.56
    }

    agent {
        kubernetes {
            inheritFrom 'my-jenkins-jenkins-agent'
            defaultContainer 'jnlp'
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins-build: app-build
spec:
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:v1.5.1-debug
      imagePullPolicy: IfNotPresent
      command: ["/busybox/cat"]
      tty: true
      volumeMounts:
        - name: jenkins-docker-cfg
          mountPath: /kaniko/.docker

    - name: kubectl
      image: lachlanevenson/k8s-kubectl
      imagePullPolicy: IfNotPresent
      command: ["/bin/sh"]
      tty: true

    - name: trivy
      image: aquasec/trivy:latest
      imagePullPolicy: IfNotPresent
      command: ["/bin/sh"]
      tty: true
      volumeMounts:
        - name: jenkins-docker-cfg
          mountPath: /root/.docker

  nodeSelector:
    gpu: missing

  tolerations:
    - key: "gpu"
      operator: "Equal"
      value: "missing"
      effect: "NoSchedule"

  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: "gpu"
                operator: In
                values: ["missing"]

  volumes:
    - name: jenkins-docker-cfg
      projected:
        sources:
          - secret:
              name: docker-credentials
              items:
                - key: .dockerconfigjson
                  path: config.json
"""
        }
    }

    parameters {
        string(name: 'TAG', defaultValue: '', description: 'Image tag to deploy (used for PROD approval)')
        string(name: 'ENV', defaultValue: '', description: 'Environment to deploy to (e.g. prod)')
    }

    stages {

        stage('Build and Publish with Kaniko') {
            when {
                not {
                    equals expected: 'prod', actual: params.ENV
                }
            }
            steps {
                container(name: 'kaniko', shell: '/busybox/sh') {
                    withEnv(['PATH+EXTRA=/busybox']) {
                        sh """
                        /kaniko/executor \
                            --dockerfile Dockerfile \
                            --context ${pwd()} \
                            --verbosity debug \
                            --destination 123456789102.dkr.ecr.us-east-1.amazonaws.com/test-deployment:${BUILD_NUMBER}
                        """
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            when {
                not {
                    equals expected: 'prod', actual: params.ENV
                }
            }
            steps {
                container('trivy') {
                    sh """
                    trivy image \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      --no-progress \
                      123456789102.dkr.ecr.us-east-1.amazonaws.com/test-deployment:${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Deploy to Dev') {
            when {
                not {
                    equals expected: 'prod', actual: params.ENV
                }
            }
            steps {
                script {
                    try {
                        sshagent (credentials: [env.SSH_CREDENTIALS]) {
                            retry(3) {
                                sh """
                                ssh -o StrictHostKeyChecking=no $REMOTE_HOST '
                                    kubectl set image deployment/test-deployment test-deployment=123456789102.dkr.ecr.us-east-1.amazonaws.com/test-deployment:${BUILD_NUMBER} -n test-dev

                                    kubectl rollout status deployment/test-deployment -n test-dev --timeout=120s

                                    kubectl rollout history deployment/test-deployment -n test-dev
                                '
                                """
                            }
                        }
                    } catch (Exception e) {

                        echo "Deployment failed. Rolling back..."

                        sshagent (credentials: [env.SSH_CREDENTIALS]) {
                            sh """
                            ssh -o StrictHostKeyChecking=no $REMOTE_HOST '
                                kubectl rollout undo deployment/test-deployment -n test-dev

                                kubectl rollout status deployment/test-deployment -n test-dev --timeout=120s
                            '
                            """
                        }

                        error("Deployment failed and rollback executed.")
                    }
                }
            }
        }

        stage('Email Committer with Approval Link') {
            when {
                not {
                    equals expected: 'prod', actual: params.ENV
                }
            }
            steps {
                script {
                    def commitId = sh(script: 'git rev-parse --short=12 HEAD', returnStdout: true).trim()
                    def commitMsg = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
                    def commitEmail = sh(script: 'git log -1 --pretty=format:%ae', returnStdout: true).trim()
                    def approveUrl = "${JENKINS_URL}/job/${JOB_NAME}/buildWithParameters?TAG=${BUILD_NUMBER}&ENV=prod"

                    emailext(
                        to: commitEmail,
                        subject: "Approval Needed: Deploy ${JOB_NAME} to PROD",
                        body: """\
Hello,

Your recent commit has been deployed to *DEV*.

- Commit ID: ${commitId}
- Commit Message: ${commitMsg}
- Build Number: ${BUILD_NUMBER}

To approve and deploy to *PROD*, click the link below:
${approveUrl}

Regards,  
Jenkins
"""
                    )
                }
            }
        }

        stage('Deploy to Prod') {
            when {
                allOf {
                    expression { return params.ENV == 'prod' }
                    expression { return params.TAG != '' }
                }
            }
            steps {
                script {
                    try {
                        sshagent (credentials: [env.SSH_CREDENTIALS]) {
                            retry(3) {
                                sh """
                                ssh -o StrictHostKeyChecking=no $REMOTE_HOST '
                                    kubectl set image deployment/test-deployment test-deployment=123456789102.dkr.ecr.us-east-1.amazonaws.com/test-deployment:${TAG} -n test-prod

                                    kubectl rollout status deployment/test-deployment -n test-prod --timeout=120s

                                    kubectl rollout history deployment/test-deployment -n test-prod
                                '
                                """
                            }
                        }
                    } catch (Exception e) {

                        echo "Production deployment failed. Rolling back..."

                        sshagent (credentials: [env.SSH_CREDENTIALS]) {
                            sh """
                            ssh -o StrictHostKeyChecking=no $REMOTE_HOST '
                                kubectl rollout undo deployment/test-deployment -n test-prod

                                kubectl rollout status deployment/test-deployment -n test-prod --timeout=120s
                            '
                            """
                        }

                        error("Production deployment failed and rollback executed.")
                    }
                }
            }
        }

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
    }

    post {
        success {
            echo "Deployment completed successfully!"
        }
        failure {
            echo "Deployment failed. Please check the logs."
        }
    }
}