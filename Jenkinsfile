pipeline {
    // 파이프라인 시작점
    // 사용할 agent 상관없도록 지정 (해당 코드의 경우 node 하나만 쓰기 때문에 상관 없도록 지정)
    agent any

    // 3분 주기로 Pipeline 구동하도록 trigger 지정
    triggers {
        pollSCM('*/3 * * * *')
    }

    // pipeline 내에서 사용할 환경변수 지정
    // aws ecs혹은 ecr을 쓰는 경우 환경변수에 접근할 수 있어야 하기 때문에 이와 같이 지정 - Jenkins도 aws를 이용하는 사용자에 속하기 때문에 aws account에서도 따로 지정을 해 주어야 한다.
    // AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY를 설정하여 aws cli명령어를 사용할 수 있도록 지정
    environment {
      AWS_ACCESS_KEY_ID = credentials('awsJenkinsId')
      AWS_SECRET_ACCESS_KEY = credentials('awsJenkinsPassword')
      AWS_DEFAULT_REGION = 'ap-northeast-2'
      HOME = '.' // Avoid npm root owned
    }

    stages {
        // Prepare Stage - 레포지토리 클론
        stage('Prepare') {
            agent any
            
            // credentialsId의 경우 깃 토큰 만들고 jenkins에 등록했을 때 이용한 id를 등록하면 됨
            steps {
                echo 'Clonning Repository'

                git url: 'https://github.com/ibasaw54/jenkins-test.git',
                    branch: 'main',
                    credentialsId: 'github-repo-token'
            }

            post {
                success {
                    echo 'Successfully Cloned Repository'
                }

                always {
                  echo "i tried..."
                }

                cleanup {
                  echo "after all other post condition"
                }
            }
        }

        // production브랜치 상에서 APP_ENV 의 Value가 production인 경우 실행되는 stage
        stage('Only for production') {
          when {
            branch 'production'
            environment name: 'APP_ENV', value: 'prod'
            anyOf {
              environment name: 'DEPLOY_TO', value: 'production'
              environment name: 'DEPLOY_TO', value: 'staging'
            }
          }
        }
        
        // aws s3 에 파일을 올림(우선 배포 과정만 빠르게 보기 위해 정적호스팅으로 간단하게 설정)
        stage('Deploy Frontend') {
          steps {
            echo 'Deploying Frontend'
            // 프론트엔드 디렉토리의 정적파일들을 S3 에 올림, 이 전에 반드시 EC2 instance profile 을 등록해야함.
            // website folder내의 파일들만 업로드하기 위해 디렉토리를 이동해주는 것
            dir ('./website'){
                sh '''
                aws s3 sync ./ s3://jenkins-test-front
                '''
            }
          }

          post {
              // If Maven was able to run the tests, even if some of the test
              // failed, record the test results and archive the jar file.
              success {
                  echo 'Successfully Cloned Repository'

                  mail  to: 'awwsb41@gmail.com',
                        subject: "Deploy Frontend Success",
                        body: "Successfully deployed frontend!"

              }

              failure {
                  echo 'I failed :('

                  mail  to: 'awwsb41@gmail.com',
                        subject: "Failed Pipelinee",
                        body: "Something is wrong with deploy frontend"
              }
          }
        }
        
        stage('Lint Backend') {
            // Docker plugin and Docker Pipeline 두개를 깔아야 사용가능
            // jenkins 서버상에 node 이미지를 띄워 lint작업을 처리하도록 하는 것
            // 실질적인 배포 환경에서는 이렇게 docker public repository에 접근하지 않고 ECR에 접근을 허용한 뒤 해당 레포지토리 내의 이미지를 사용할 것
            agent {
              docker {
                image 'node:latest'
              }
            }
            
            steps {
              dir ('./server'){
                  sh '''
                  npm install&&
                  npm run lint
                  '''
              }
            }
        }
        
        stage('Test Backend') {
          agent {
            docker {
              image 'node:latest'
            }
          }
          steps {
            echo 'Test Backend'

            dir ('./server'){
                sh '''
                npm install
                npm run test
                '''
            }
          }
        }
        
        // 도커를 빌드하여 배포한다. 이 과정에서 해당 Jenkins node에는 도커가 설치되어 있어야 함
        // --build-arg env=${PROD} -> 멀티플랫폼 배포 환경의 경우를 상정함
        stage('Bulid Backend') {
          agent any
          steps {
            echo 'Build Backend'

            dir ('./server'){
                sh """
                docker build . -t server --build-arg env=${PROD}
                """
            }
          }
          // 서버를 빌드하다 실패했는데 배포가 실행되면 안됨 -> error를 뱉고 나머지 Pipeline 종료
          post {
            failure {
              error 'This pipeline stops here...'
            }
          }
        }
        
        // 실 환경에서는 ECS업데이트 과정이 포함되어 있어야 함
        stage('Deploy Backend') {
          agent any

          // 기존 이미지를 삭제하고 Build 단계에서 생성한 이미지를 실행시키는 과정
          // 즉 N차 배포에서는 docker rm -f $(docker ps -aq) 명령어 필요
          steps {
            echo 'Run Backend'

            dir ('./server'){
                sh '''
                docker run -p 80:80 -d server
                '''
            }
          }

          post {
            success {
              mail  to: 'awwsb41@gmail.com',
                    subject: "Deploy Success",
                    body: "Successfully deployed!"
                  
            }
          }
        }
    }
}
// push test