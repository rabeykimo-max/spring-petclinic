pipeline {
    agent { label 'wazuh-dashboard' }

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-amd64'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"

        HARBOR_REGISTRY = '172.31.25.149:30002'
        HARBOR_PROJECT  = 'devops-lab'
        APP_NAME        = 'petclinic'
        IMAGE_NAME      = "${HARBOR_REGISTRY}/${HARBOR_PROJECT}/${APP_NAME}"

        GITOPS_REPO = 'https://github.com/rabeykimo-max/argocd-lab.git'
        GITOPS_FILE = 'apps/petclinic/overlays/dev/kustomization.yaml'
    }

    stages {

        stage('Checkout Source') {
            steps {
                deleteDir()

                git branch: 'main',
                    credentialsId: 'github-gitops',
                    url: 'https://github.com/rabeykimo-max/spring-petclinic.git'
            }
        }

        stage('Environment Check') {
            steps {
                sh '''
                    echo "===== JAVA ====="
                    java -version
                    javac -version

                    echo "===== MAVEN ====="
                    ./mvnw --version
                '''
            }
        }

        stage('Compile') {
            steps {
                sh '''
                    echo "===== MAVEN COMPILE ====="
                    ./mvnw clean compile
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    echo "===== MAVEN TEST ====="
                    ./mvnw test
                '''
            }

            post {
                always {
                    junit allowEmptyResults: true,
                          testResults: 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package') {
            steps {
                sh '''
                    echo "===== MAVEN PACKAGE ====="
                    ./mvnw package -DskipTests

                    echo "===== JAR ====="
                    ls -lh target/*.jar
                '''

                archiveArtifacts artifacts: 'target/*.jar',
                                 fingerprint: true
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "===== DOCKER BUILD ====="
                    echo "Image: ${IMAGE_NAME}:${BUILD_NUMBER}"

                    docker build \
                      -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Harbor Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'harbor-credentials',
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$HARBOR_PASSWORD" | \
                        docker login "$HARBOR_REGISTRY" \
                          -u "$HARBOR_USER" \
                          --password-stdin
                    '''
                }
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                    echo "===== PUSH IMAGE ====="

                    docker push \
                      ${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }
        }

        stage('Checkout GitOps') {
            steps {
                dir('gitops') {
                    git branch: 'main',
                        credentialsId: 'github-gitops',
                        url: "${GITOPS_REPO}"
                }
            }
        }

        stage('Update DEV GitOps') {
            steps {
                dir('gitops') {

                    sh '''
                        set -e

                        echo "===== CURRENT DEV ====="
                        cat "${GITOPS_FILE}"

                        echo "===== UPDATE IMAGE TAG ====="

                        sed -i \
                          "s/newTag: .*/newTag: \\"${BUILD_NUMBER}\\"/" \
                          "${GITOPS_FILE}"

                        echo "===== NEW DEV ====="
                        cat "${GITOPS_FILE}"

                        echo "===== VALIDATE ====="

                        grep \
                          "newTag: \\"${BUILD_NUMBER}\\"" \
                          "${GITOPS_FILE}"

                        kubectl kustomize \
                          apps/petclinic/overlays/dev \
                          >/tmp/petclinic-dev.yaml

                        grep \
                          "image: ${IMAGE_NAME}:${BUILD_NUMBER}" \
                          /tmp/petclinic-dev.yaml
                    '''
                }
            }
        }

        stage('Push GitOps') {
            steps {
                dir('gitops') {

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'github-gitops',
                            usernameVariable: 'GIT_USER',
                            passwordVariable: 'GIT_TOKEN'
                        )
                    ]) {

                        sh '''
                            set -e

                            git config user.name "Jenkins"
                            git config user.email "jenkins@local"

                            git add "${GITOPS_FILE}"

                            if git diff --cached --quiet; then
                                echo "No GitOps change."
                                exit 0
                            fi

                            git commit \
                              -m "Deploy petclinic:${BUILD_NUMBER} to dev"

                            git push \
                              "https://${GIT_USER}:${GIT_TOKEN}@github.com/rabeykimo-max/argocd-lab.git" \
                              HEAD:main
                        '''
                    }
                }
            }
        }
    }

    post {

        success {
            echo "================================="
            echo "CI/CD SUCCESS"
            echo "Image: ${IMAGE_NAME}:${BUILD_NUMBER}"
            echo "Environment: DEV"
            echo "================================="
        }

        failure {
            echo "CI/CD FAILED"
        }

        always {
            sh '''
                docker logout ${HARBOR_REGISTRY} || true
            '''
        }
    }
}
