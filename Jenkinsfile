pipeline {
    agent any
    
    environment{
        NETLIFY_SITE_ID = 'a6340c47-39b7-4938-8c7b-caa6bd70449b'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.${BUILD_ID}"
    }

    stages {
        stage('Docker'){
            steps{
                sh 'docker build -t e2e-playwright .'
            }
        }

        stage('Build') {
            agent{
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }

        stage('Test'){
            parallel{
                stage('Unit Test'){
                    agent{
                        docker{
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps{
                        sh '''
                            npm test
                        '''
                    }
                    post{
                        always{
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('Local E2E Test'){
                    agent{
                        docker{
                            image 'e2e-playwright'
                            reuseNode true
                        }
                    }
                    steps{
                        sh '''
                            serve -s build &
                            sleep 15
                            npx playwright test --reporter=html
                        '''
                    }
                    post{
                        always{
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Local E2E Test Report', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy staging'){
            agent{
                docker{
                    image 'e2e-playwright'
                    reuseNode true
                }
            }
            environment{
                CI_ENVIRONMENT_URL = "undefined"
            }
            steps{
                sh '''
                    netlify --version
                    echo "Deploying to prod. Site id: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --json > netlify-deploy-info.json
                    CI_ENVIRONMENT_URL=$(npx node-jq -r '.deploy_url' netlify-deploy-info.json)
                    sleep 15
                    npx playwright test --reporter=html
                '''
            }
            post{
                always{
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Staging E2E Test Report', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        stage('Deploy prod'){
            agent{
                docker{
                    image 'e2e-playwright'
                    reuseNode true
                }
            }
            environment{
                CI_ENVIRONMENT_URL = 'https://spontaneous-bombolone-b271df.netlify.app'
            }
            steps{
                sh '''
                    npm i netlify-cli
                    netlify --version
                    echo "Deploying to prod. Site id: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --prod
                    sleep 15
                    npx playwright test --reporter=html
                '''
            }
            post{
                always{
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Prod E2E Test Report', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }
}
