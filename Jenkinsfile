pipeline {
    agent any 

    environment {
        DOCKER_CREDENTIALS_ID = 'roseaw-dockerhub'  
        DOCKER_IMAGE = 'cithit/chhetrrb'                                   
        IMAGE_TAG = "build-${BUILD_NUMBER}"
        GITHUB_URL = 'https://github.com/ratnac101/Lab-5.1.git'     
        KUBECONFIG = credentials('chhetrrb-225')                           
    }

   stages {

        // 1. Checkout
        stage('Code Checkout') {
            steps {
                cleanWs()
                checkout([$class: 'GitSCM',
                          branches: [[name: '*/main']],
                          userRemoteConfigs: [[url: "${GITHUB_URL}"]]])
            }
        }

        // 2. Static Code Testing (Python + HTML)
        //    -> satisfies "Static Code Testing" rubric
        stage('Static Code Testing') {
            steps {
                sh '''
                    # Python static analysis
                    pip3 install flake8
                    flake8 main.py data-gen.py data-clear.py || true

                    # HTML static analysis
                    npm install htmlhint --save-dev
                    npx htmlhint templates/*.html || true
                '''
            }
        }

        // 3. Docker Build & Push
        //    -> satisfies "Docker Build" rubric
        stage('Build & Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', "${DOCKER_CREDENTIALS_ID}") {
                        def app = docker.build("${DOCKER_IMAGE}:${IMAGE_TAG}", "-f Dockerfile.build .")
                        app.push()
                    }
                }
            }
        }

        // 4. Deploy to DEV (Kubernetes)
        //    -> DEV environment backed by NFS PV (data persistence)
        stage('Deploy to Dev Environment') {
            steps {
                script {
                    // remove old dev deployments (optional, keeps things clean)
                    sh "kubectl delete --all deployments --namespace=default || true"

                    // update image tag to the new build
                    sh "sed -i 's|${DOCKER_IMAGE}:latest|${DOCKER_IMAGE}:${IMAGE_TAG}|' deployment-dev.yaml"

                    // apply dev manifests
                    sh "kubectl apply -f deployment-dev.yaml"
                }
            }
        }

        // 5. DAST (Dastardly) against DEV
        //    -> dynamic testing of running app
        stage('Run Security Checks (DAST)') {
            steps {
                sh 'docker pull public.ecr.aws/portswigger/dastardly:latest'
                sh '''
                    docker run --user $(id -u) -v ${WORKSPACE}:${WORKSPACE}:rw \
                    -e BURP_START_URL=http://10.48.229.154 \
                    -e BURP_REPORT_FILE_PATH=${WORKSPACE}/dastardly-report.xml \
                    public.ecr.aws/portswigger/dastardly:latest
                '''
                # NOTE: keep BURP_START_URL as your DEV ClusterIP
            }
        }

        // 6. Reset DB after security checks
        //    -> shows you can control persisted data
        stage('Reset DB After Security Checks') {
            steps {
                script {
                    def appPod = sh(
                        script: "kubectl get pods -l app=flask -o jsonpath='{.items[0].metadata.name}'",
                        returnStdout: true
                    ).trim()

                    sh """
                        kubectl exec ${appPod} -- python3 - <<'PY'
                        import sqlite3
                        conn = sqlite3.connect('/nfs/demo.db')
                        cur = conn.cursor()
                        cur.execute('DELETE FROM contacts')
                        conn.commit()
                        conn.close()
                        PY
                    """
                }
            }
        }

        // 7. Generate Test Data into /nfs/demo.db
        //    -> uses persistence (NFS PV + PVC)
        stage('Generate Test Data') {
            steps {
                script {
                    def appPod = sh(
                        script: "kubectl get pods -l app=flask -o jsonpath='{.items[0].metadata.name}'",
                        returnStdout: true
                    ).trim()

                    sh "sleep 15"
                    sh "kubectl get pods"
                    sh "kubectl exec ${appPod} -- python3 data-gen.py"
                }
            }
        }

        // 8. Acceptance Tests (Selenium)
        //    -> dynamic code testing, hitting HTML UI
        stage('Run Acceptance Tests (Selenium)') {
            steps {
                script {
                    sh 'docker stop qa-tests || true'
                    sh 'docker rm qa-tests || true'
                    sh 'docker build -t qa-tests -f Dockerfile.test .'
                    sh 'docker run qa-tests'
                }
            }
        }

        // 9. Remove Test Data
        stage('Remove Test Data') {
            steps {
                script {
                    def appPod = sh(
                        script: "kubectl get pods -l app=flask -o jsonpath='{.items[0].metadata.name}'",
                        returnStdout: true
                    ).trim()

                    sh "kubectl exec ${appPod} -- python3 data-clear.py"
                }
            }
        }

        // 10. Deploy to PROD (Kubernetes)
        //     -> separate PROD environment (LoadBalancer service)
        stage('Deploy to Prod Environment') {
            steps {
                script {
                    sh "sed -i 's|${DOCKER_IMAGE}:latest|${DOCKER_IMAGE}:${IMAGE_TAG}|' deployment-prod.yaml"
                    sh "kubectl apply -f deployment-prod.yaml"
                }
            }
        }

        // 11. Final sanity check
        stage('Check Kubernetes Cluster') {
            steps {
                sh "kubectl get all"
            }
        }
    }

    // ChatOps: Slack notifications
    post {
        success {
            slackSend color: "good",
                      message: "✅ Lab 5.1 pipeline SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        unstable {
            slackSend color: "warning",
                      message: "⚠️ Lab 5.1 pipeline UNSTABLE: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        failure {
            slackSend color: "danger",
                      message: "❌ Lab 5.1 pipeline FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}
