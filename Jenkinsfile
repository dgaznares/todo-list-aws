pipeline{
    agent any
    environment {
    
        DEVELOP_ENVIRONMENT="develop"
        MASTER_ENVIRONMENT="master"
        gitBranch = "${env.GIT_BRANCH}";
        STAGE_DEVELOP = "staging"
        STAGE_MASTER =  "production"
        STACK_NAME_DEVELOP = "staging-todo-list-aws"
        STACK_NAME_MASTER = "production-todo-list-aws"
        ENVIRONMENT= ""
    }
    stages{
        stage('Get Code'){
            steps{
                cleanWs()
                echo 'GetCode stage'
                sh 'env'
                script{
                
                    def gitBranchLowecase = gitBranch.toLowerCase();
                    if (gitBranchLowecase.contains(DEVELOP_ENVIRONMENT)
                        ENVIRONMENT=DEVELOP_ENVIRONMENT
                    else 
                    if(gitBranchLowecase.contains(DEVELOP_ENVIRONMENT)
                        ENVIRONMENT=MASTER_ENVIRONMENT
                }

                withCredentials([string(credentialsId: 'GitHub-TOKEN', variable: 'TOKEN')]) {
                    // Ahora puedes usar la variable TOKEN de forma segura
                    sh 'echo "Usando el token: $TOKEN"'
                    echo "Getting code from ${ENVIRONMENT}..."
                    git branch: "${ENVIRONMENT}", url: 'https://${TOKEN}@github.com/dgaznares/todo-list-aws.git'
                    sh 'ls -la'
                }
            }
        }
        stage('Setup Python Environment') { // Nueva etapa para configurar el entorno
            steps {
                  sh '''
                    echo 'Setting up Python virtual environment and installing dependencies...'
                    if [ ! -d "venv" ]; then
                        echo "Python virtual environment is NOT installed. Installing ..."
                        python3 -m venv venv
                        ./venv/bin/pip install --upgrade pip
                        ./venv/bin/pip install flake8 bandit pytest requests
                    else
                        echo "Python virtual environment is already installed."
                    fi
                '''
            }
        }
        stage('Static Test'){
            when {
                expression {return ENVIRONMENT == 'develop'}
                //branch 'develop' // Esta stage solo se ejecuta si la rama actual es 'develop'
            }
            steps{
                 echo 'Static test stage'
                 sh '''
                    . venv/bin/activate && flake8 --exit-zero --format=pylint src > flake8.out
                 '''
                 recordIssues tools: [
                        flake8(linesLookAhead: 0, pattern: 'flake8.out')
                 ]
                 sh '''
                    . venv/bin/activate && bandit --exit-zero -r src -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                  '''
                 recordIssues tools: [
                     pyLint(name: "Bandit", pattern: 'bandit.out')
                 ]
            }
        }
        stage('Deploy'){
            steps{
                 echo "Deploy stage from ${ENVIRONMENT}..."
                 sh '''
                    sam build
                    sam validate --lint
                 '''
                 script{
                     def stageName = STAGE_DEVELOP
                     if (ENVIRONMENT =='master'){
                         stageName = STAGE_MASTER
                     }
                    sh " sam deploy --config-env ${stageName} --no-confirm-changeset --no-fail-on-empty-changeset"
                 }
            }
        }
        stage('Rest Test'){
            steps{
                 echo "Rest Test stage ${ENVIRONMENT}"
                 script {
                    def stackName = STACK_NAME_DEVELOP 
                    if (ENVIRONMENT == "master") {
                        stackName = STACK_NAME_MASTER
                    }
                    def baseUrl = sh(
                                    script: """
                                                aws cloudformation describe-stacks \
                                                    --stack-name ${stackName} \
                                                    --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' \
                                                    --region us-east-1 \
                                                    --output text
                                            """
                                    ,
                                    returnStdout: true
                                ).trim()
                    echo "baseUrl -> ${baseUrl}"
                    env.BASE_URL = baseUrl
                    echo "la BASE_URL del entorno ${ENVIRONMENT} es : ${env.BASE_URL}"
                 }
                 echo 'Lanzamos el pytest'
                 sh ". venv/bin/activate && pytest --junitxml=result-rest.xml ./test/integration/todoApiTest.py -m ${ENVIRONMENT}"
                 echo 'Recogemos las pruebas del pytest'
                 junit 'result-rest.xml'
            }
        }
        stage('Promote'){
            when {
                expression {return ENVIRONMENT == 'develop'}
                //branch 'develop' // Esta stage solo se ejecuta si la rama actual es 'develop'
            }
            steps{
                 echo 'Promote stage'
                 withCredentials([string(credentialsId: 'GitHub-TOKEN', variable: 'TOKEN')]) {
                    sh '''
                        git checkout master
                        git merge develop
                        git push https://${TOKEN}@github.com/dgaznares/todo-list-aws.git
                    '''
                 }
            }
        }
    }
}