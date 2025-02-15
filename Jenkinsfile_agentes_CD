pipeline {
    agent any
    stages {
        stage('Get Code') {
            steps {
               // Se obtiene el codigo de GitHub
                git branch: 'develop', url: 'https://github.com/Gerardo-j-Gonzalez/todo-list-aws.git'
                echo "El workspace actual es: ${env.WORKSPACE}"
                stash name:'code', includes:'**'
            }
        }
        stage('Deploy') {
            agent {label 'principal'} 
            steps {
                script {
                    unstash name:'code'

					echo "Comprobacion de whoami y hostname en Deploy"
					sh 'whoami'
					sh 'hostname'

                    echo "Se realiza el despliegue con AWS SAM"
                    
                    // Se realiza la build del codigo
                    sh 'sam build' 
                    
                    // Se hace la validacion de SAM
                    sh 'sam validate --region us-east-1' 
                    
                    // Desplegamos utilizando la configuracion 'staging' que esta en samconfig.toml 
                    sh '''
                        sam deploy \
                            --config-env production  \
                            --no-confirm-changeset --no-fail-on-empty-changeset
                    '''
                    
                    echo "Despliegue correcto" 
                }
            }
        }    
        stage('Rest Test'){
            agent {label 'Agente_CP1_4-1'}  
            steps{
                script {
                    unstash name:'code'

					echo "Comprobacion de whoami y hostname en Rest Test"
					sh 'whoami'
					sh 'hostname'

                    echo "Obtenemos la URL del API desde CloudFormation" 
 
                    // Obtenemos `BASE_URL` desde los outputs de CloudFormation
                    def BASE_URL = sh(
                        script: """
                            aws cloudformation describe-stacks --stack-name todo-list-aws-production --region us-east-1 \
                                --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --output text
                            """,
                            returnStdout: true
                    ).trim()
 
                    // Si no se obtiene `BASE_URL` , se para el pipeline 
                    if (BASE_URL == "" || BASE_URL == "None") {
                        error "No se pudo obtener la URL del API. Verifica que el stack se haya desplegado correctamente." 
                    }
 
                    echo "API URL obtenida: ${BASE_URL}" 
 
                    // Definimos `BASE_URL` en el entorno del pipeline 
                    withEnv(["BASE_URL=${BASE_URL}"]) {
                        echo "Se ejecutan las pruebas REST con Pytest" 
                        
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            sh '''
                                python3 -m pytest --version
                                python3 -m pytest -s -v -k "listtodos or gettodo" --junitxml=result-integration.xml test/integration/todoApiTest.py | tee result-integration.log
                            '''
                        }
                        
                        echo "Se comrueba el resultado buscando en el log" 
                        def passedTests = sh(
                            script: "grep -c 'PASSED' result-integration.log || echo 0",
                            returnStdout: true
                        ).trim()
 
                        if (passedTests != "2") {
                            error "No se han superado todas las pruebas. Se necesitan 2 PASSED, pero se encontraron: ${passedTests}" 
                        }
 
                        echo "Todas las pruebas se han superado correctamente (${passedTests}/2)." 
                        echo "Se envian los resultados de las pruebas a XML para verlo en JENKINS" 
                        
                        junit 'result-integration.xml' 
                    }

                }
            }
        }
    }
}