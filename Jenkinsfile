pipeline {
    agent any

    stages {
        /*stage('Build') {
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
        } */
        stage('Test'){
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
        }
    }
    post {
            always {
                junit 'junit-results/junit.xml'
            }
    }
}
