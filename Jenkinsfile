pipeline {
    agent any
    
    environment{
        NETLIFY_SITE_ID = 'a6340c47-39b7-4938-8c7b-caa6bd70449b'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    }

    stages {
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

                stage('E2E Test'){
                    agent{
                        docker{
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }
                    steps{
                        sh '''
                            npx serve -s build &
                            sleep 15
                            npx playwright test --reporter=html
                        '''
                    }
                    post{
                        always{
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy staging'){
            agent{
                docker{
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps{
                sh '''
                    npm i netlify-cli
                    npx netlify-cli --version
                    echo "Deploying to prod. Site id: $NETLIFY_SITE_ID"
                    npx netlify-cli status
                    npx netlify-cli deploy --dir=build
                '''
            }
        }

        stage('Approval'){
            agent any

            steps{
                timeout(time: 1, unit: 'HOURS') {
                    input message: 'Ready to deploy to production environment? ', ok: 'Yes, proceed with the deployment'
                }
            }
        }
        
        stage('Deploy prod'){
            agent{
                docker{
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps{
                sh '''
                    npm i netlify-cli
                    npx netlify-cli --version
                    echo "Deploying to prod. Site id: $NETLIFY_SITE_ID"
                    npx netlify-cli status
                    npx netlify-cli deploy --dir=build --prod
                '''
            }
        }

        stage('Prod E2E Test'){
            agent{
                docker{
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }
            environment{
                CI_ENVIRONMENT_URL = 'https://spontaneous-bombolone-b271df.netlify.app'
            }
            steps{
                sh '''
                    npx playwright test --reporter=html
                '''
            }
            post{
                always{
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright E2E Prod HTML Report', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }
}
