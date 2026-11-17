pipeline {
    agent {
        docker {
            image 'node:26-alpine'
        }
    }

    environment {
        CI = 'true'
        NPM_CONFIG_CACHE = '/tmp/.npm'
    }

    stages {
        stage('Dependências') {
            steps {
                sh 'node -v && npm -v'
                sh 'npm ci'
            }
        }

        stage('Qualidade e Tipagem') {
            parallel {
                stage('Lint & Code Pattern (Biome)') {
                    steps {
                        sh 'npm run check:ci'
                    }
                }
                stage('Type Check (TSC)') {
                    steps {
                        sh 'npm run typecheck'
                    }
                }
            }
        }

        stage('Testes Unitários (Vitest)') {
            steps {
                sh 'npm test'
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
    }

    post {
        success {
            echo 'Pipeline concluído com sucesso!'
        }
        failure {
            echo 'Falha na esteira. Verifique o Console Output.'
        }
        always {
            cleanWs()
        }
    }
}
