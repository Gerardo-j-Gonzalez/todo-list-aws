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
        stage('Static Test') {
            agent {label 'Agente_CP1_4-1'}     
            steps {
                script {
                    unstash name:'code'

					echo "Comprobacion de whoami y hostname en Static Test"
					sh 'whoami'
					sh 'hostname'

                    echo "Se realiza el analisis estatico con Flake8 y Bandit" 
                    // Ejecuta Flake8 y genera informe 
                    sh 'flake8 --exit-zero --format=pylint src >flake8.out' 
                    // Ejecuta Bandit y genera informe 
                    sh 'bandit --exit-zero -r . -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"|| true' 
                    echo "Generando los ficheros de los informes" 
                    recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')] 
                    recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')] 

                }
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

                    echo "Se realiza el despliegue con AWS SAM..." 
                    // Se realiza la build del codigo
                    sh 'sam build' 
                    // Se hace la validacion de SAM
                    sh 'sam validate --region us-east-1' 
                    
					// Desplegamos utilizando la configuracion 'staging' que esta en samconfig.toml 

                    sh '''
                    sam deploy \
                        --config-env staging \
                        --parameter-overrides CreateDynamoTable=false \
                        --no-confirm-changeset \
                        --no-fail-on-empty-changeset
                    '''
                    echo "Despliegue correcto" 


                }
            }
        }    
        stage('Rest Test'){
            agent {label 'Agente_CP1_4-2'}  
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
                            aws cloudformation describe-stacks --stack-name todo-list-aws-staging --region us-east-1 \
                                --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --output text 
                        """, 
                        returnStdout: true 
                    ).trim() 
 
                    // Si no se obtiene `BASE_URL` , se para el pipeline 
                    if (BASE_URL == "" || BASE_URL == "None") { 
                        error "No se pudo obtener la URL del API. Verifica que el stack se haya desplegado correctamente." 
                    } 



                    echo "API URL obtenida: ${BASE_URL}" 

                    withEnv(["BASE_URL=${BASE_URL}"]) { 
                        echo "Se ejecutan las pruebas REST con Pytest" 
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') { 
                            sh ''' 
                                /usr/bin/python3 -m pytest --version 
                                /usr/bin/python3 -m pytest -s -v --junitxml=result-integration.xml test/integration/todoApiTest.py | tee result-integration.log 
                            ''' 


                        } 

                        echo "Se envian los resultados de las pruebas a XML para verlo en JENKINS" 
                        junit 'result-integration.xml' 
                    }


                }
            }
        }
 
       
        stage('Promote') {
            steps {
                script {
                    unstash name:'code'
                    echo "Promocion de la version a MASTER" 

					echo "Comprobacion de whoami y hostname en Promote"
					sh 'whoami'
					sh 'hostname'
					
                    // Obtenemos el token desde los secretos de Jenkins 
                    withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'TOKEN')]) { 
                        // Configuramos el usuario de Git
                        sh ''' 
                            git config --global user.email "gerardo_j_gonzalez@hotmail.com" 
                            git config --global user.name "gerardo-j-gonzalez" 
                        ''' 
                        
						// Cambiamos la URL del repositorio para usar el token 
                        sh ''' 
                            git remote set-url origin https://$TOKEN@github.com/gerardo-j-gonzalez/todo-list-aws.git 
                        ''' 
                        
						// Comprobamos que estamos en la rama develop
                        def currentBranch = sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim() 
                        if (currentBranch != "develop") { 
                            error "La rama actual es '${currentBranch}', pero se necesita la rama develop." 
                        } 

                        echo "Cambiamos a la rama master" 
						
                        sh ''' 
                            git fetch origin 
                            git checkout master 
                            git pull --rebase origin master 
                        ''' 
 
                        echo "Mergeamos de la rama develop a la rama master" 

                        sh ''' 
                            git merge --no-ff develop -m "Mergeo automatico desde develop a master [Jenkins CI]" || true 
                            git checkout --ours CHANGELOG.md # Mantener version de develop 
                            git add CHANGELOG.md 
                            git commit -m "Resolvemos el conflicto en CHANGELOG.md manteniendo la version de develop" || true 
                        ''' 
 
                        echo "Subimos los cambios a GitHub con push" 
                        sh ''' 
                            git push origin master 
                        ''' 
 
                        echo "Promocion a MASTER completada" 


                    }
                }
            }
        } 
    }
}