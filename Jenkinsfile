pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '1fff8fc0-39ca-457a-95ae-91081bc6a1a8' // netlify 사이트 아이디를 저장하는 로컬 환경 변수
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID" // https://www.jenkins.io/doc/book/pipeline/jenkinsfile/#using-environment-variables
        AWS_S3_BUCKET = 'learn-jenkins-lewisjlee-20250702'
    }

    // 애플리케이션 빌드
    stages {
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
                            image 'my-node-playwright'
                            reuseNode true
                        }
                    }
                    steps{
                        echo 'Test stage'
                        sh '''
                            serve -s build &
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
            // E2E 테스트를 수행할 URL 환경 변수, environment 블록으로 사전 선언해야 playwright가 인식
            environment{
                CI_ENVIRONMENT_URL = "my-stage-url"
            }

            steps{
                echo 'Test stage'
                sh '''
                    netlify --version
                    echo "Deploying to production. Site ID : $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --json > json-output.json # Stage용 임시 환경에 배포 및 json 파일 추출
                    CI_ENVIRONMENT_URL=$(jq -r '.deploy_url' json-output.json) # 추출한 json 파일의 deploy_url의 value 출력
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
        stage('Deploy to Prod(S3)'){
            agent {
                docker {
                    image 'amazon/aws-cli'
                    args "--entrypoint=''"
                    reuseNode true
                }
            }
            environment{
                AWS_S3_BUCKET = "${env.AWS_S3_BUCKET}"
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-jenkins', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh '''
                        aws --version
                        aws s3 sync build s3://$AWS_S3_BUCKET
                    '''
                }           
            }
        }
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
                CI_ENVIRONMENT_URL = "http://${env.AWS_S3_BUCKET}.s3-website.ap-northeast-2.amazonaws.com"
            }

            steps{
                echo 'Test stage'
                sh '''
                    node --version
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
