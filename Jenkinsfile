pipeline {
    agent any

    environment {
        //Netlify is a simple cloudhosting platform for web apps
        NETLIFY_SITE_ID = 'bf83a7aa-5739-42bd-a1a2-f6e9b04d3db8'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
    }

    stages {

        // Builds the docker container worker and installs everything needed including node moduless
        stage('Build') {
            agent{
                docker {
                    image 'node:18-alpine'
                    //allows the agent to be reused for multiple stages
                    reuseNode true
                }
            }
            steps {
                //Printing out values to show that node and npm are available in the container
                // As well as building out what is in package.jsonsss
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

        // Testing the build/Web App withn a Unit & E2E tests in parallel
        stage('Tests') {
            //runs the 2 different test stages in parallel
            parallel{

                stage('Unit Tests'){
                    agent{
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps{
                        echo 'Test stage'
                        //Testing if index.html exists in the build folder
                        //using npm test runs the test scripts located in package.json file
                        sh '''
                            test -f build/index.html
                            npm test
                        '''
                    }
                    post{
                        // Always save the j-Unit Test Results
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('E2E'){
                    agent{
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }
                    steps{
                        echo 'E2E Test stage'
                        // Installs Serve which is a lightweight tool to preview static websites locally
                        // Then uses Playwright to generate an HTML report of the test result
                        sh '''
                            npm install serve
                            node_modules/.bin/serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post{
                        //Always publish the Playwright report whether it passed or failed
                        //playright report checks to make sure all elements on the webpage exist
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Local', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }

            }
        }

        // Deploy the Web App to a NON-prod Environment
        stage('Deploy staging'){
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = "STAGING_URL_TO_BE_SET"
            }
            steps {
                // Netlify is a simple cloudhosting platform for web apps
                // Install Netlify and use the local/non-global version
                // Use Netlify command to deploy and save the outpu as a json file
                // Read from the json to set the environment variable with the dynamic NON-Prod staging URL
                // When Enviro Var is set, the playwright test will use that URL to generate a HTML report
                sh '''
                    npm install netlify-cli node-jq
                    node_modules/.bin/netlify --version
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json
                    # Below sets the dynamic NON-Prod URL the enivro variable (by reading the value from json) which playwright test will use to generate a html report
                    CI_ENVIRONMENT_URL=$(node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json)

                    sleep 2
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Staging E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        // Asks for approval before deploying to prod, One can check the NON-prod Environment playwright Test report...
        // and anything else to ensure it is working first b4 approval
        stage('Approval') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Do you wish to deploy to production?', ok: 'Yes, I am sure!"'
                }
            }
        }
        
        // Deploy the Web App to prod if approved
        stage('Deploy prod'){
            agent{
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            environment {
                //Proper Web App URL in prod
                CI_ENVIRONMENT_URL = 'https://heroic-kheer-2d4aed.netlify.app'
            }

            steps {
                // Install Netlify and use the local/non-global version
                // Use Netlify to deploy with the --prod flag
                // Playwright test will use the Enviro URL to generate a HTML report after it is deployed
                sh '''
                    node --version
                    npm install netlify-cli 
                    node_modules/.bin/netlify --version
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod

                    sleep 2
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Prod E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

    }
}
