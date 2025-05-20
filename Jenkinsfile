pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '1fff8fc0-39ca-457a-95ae-91081bc6a1a8' // netlify 사이트 아이디를 저장하는 로컬 환경 변수
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    }

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
                    ls -al
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -al
                '''
            }
        }
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
                                publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright HTML Report', reportTitles: '', useWrapperFileDirectly: true])
                            }
                    }
                }
            }
        }

        stage('Deploy') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install netlify-cli@20.1.1
                    node_modules/.bin/netlify --version
                    echo "Deploying to production. Project ID : $NETLIFY_PROJECT_ID"
                    node_modules/.bin/netlify status
                '''
            }
        }
    }
}
