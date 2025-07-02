pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '1fff8fc0-39ca-457a-95ae-91081bc6a1a8' // netlify 사이트 아이디를 저장하는 로컬 환경 변수
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID" // https://www.jenkins.io/doc/book/pipeline/jenkinsfile/#using-environment-variables
    }

    // 애플리케이션 빌드
    stages {
        stage('Docker image build'){
            steps{
                sh 'docker build -t my-node-playwright .'
            }
        }

        stage('Build') {
            agent {
                docker { // 에이전트에 nodejs 컨테이너 실행
                    image 'node:18-alpine'
                    reuseNode true // 모든 스테이지에 동일한 workspace 사용
                }
            }
            steps {
                sh '''
                    echo "small changes"
                    ls -al
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -al
                '''
            }
        }

        // local 테스트 병렬 실행
        stage('Tests'){
            parallel {
                stage('Unit test'){
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }
                    steps{
                        echo 'Test stage'
                        sh '''
                            test -f build/index.html # index.html 파일이 존재하는지 확인
                            npm test # 소스코드 테스트
                        '''
                    }
                    post {
                        always {
                            junit 'junit-results/junit.xml'
                        }
                    }
                }

                stage('E2E'){
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.52.0-noble'
                            reuseNode true
                        }
                    }
                    steps{
                        echo 'Test stage'
                        sh '''
                            npm install serve
                            node_modules/.bin/serve -s build &
                            sleep 10 # 앞선 빌드가 완료될 때까지 대기하는 시간
                            npx playwright test --reporter=html # E2E Test 수행
                        '''
                    }
                    post {
                        always {
                            // Playwright E2E 테스트에 대한 HTML Report
                                publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Local', reportTitles: '', useWrapperFileDirectly: true])
                            }
                    }
                }
            }
        }

        // Prod 배포 전 Stage 환경에 배포 및 Stage E2E
        stage('Deploy to Stage and E2E'){
            agent {
                docker {
                    image 'my-node-playwright'
                    reuseNode true
                }
            }
/*            // E2E 테스트를 수행할 Prod 환경 URL
            environment{
                CI_ENVIRONMENT_URL = "${env.STAGE_URL}"
            }
*/

            steps{
                echo 'Test stage'
                sh '''
                    netlify --version
                    echo "Deploying to production. Site ID : $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --json > json-output.json # Stage용 임시 환경에 배포 및 json 파일 추출
                    CI_ENVIRONMENT_URL=$(node-jq -r '.deploy_url' json-output.json) # 추출한 json 파일의 deploy_url의 value 출력
                    npx playwright test --reporter=html # E2E Test 수행
                '''
            }
            post {
                always {
                    // Playwright E2E 테스트에 대한 HTML Report
                        publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Stage E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

/*        // Prod 배포 전 최종 검토 및 승인
        stage('Approval to deploy to Prod'){
            steps{
                timeout(time: 1, unit: 'HOURS') {
                input cancel: 'No', message: 'Prod 환경에 최종 배포합니다.', ok: 'Yes'
                }
            }
        }
*/

        // Prod 환경에 배포 및 E2E 테스트
        stage('Deploy to Prod and E2E'){
            agent {
                docker {
                    image 'my-node-playwright'
                    reuseNode true
                }
            }
            // E2E 테스트를 수행할 Prod 환경 URL
            environment{
                CI_ENVIRONMENT_URL = 'https://fluffy-yeot-2c6d6e.netlify.app'
            }

            steps{
                echo 'Test stage'
                sh '''
                    node --version
                    netlify --version
                    echo "Deploying to production. Site ID : $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --prod # Prod 환경의 build 디렉토리에 배포
                    npx playwright test --reporter=html # E2E Test 수행
                '''
            }
            post {
                always {
                    // Playwright E2E 테스트에 대한 HTML Report
                        publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Prod E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }
}
