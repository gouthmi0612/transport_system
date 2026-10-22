
pipeline {
    agent any

    parameters {
        booleanParam(
            name: 'DEPLOY',
            defaultValue: true,
            description: 'Deploy the application to Kubernetes'
        )

        booleanParam(
            name: 'SIMULATE_BAD_RELEASE',
            defaultValue: false,
            description: 'Test automatic rollback'
        )
    }

    environment {
        IMAGE = 'transport-dashboard'
        DOCKERHUB_USERNAME = 'gouthmi06'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Parallel Validation') {
            parallel {
                stage('Check HTML') {
                    steps {
                        powershell '''
                            if (!(Test-Path "index.html")) {
                                throw "index.html is missing"
                            }

                            $html = Get-Content "index.html" -Raw

                            if ($html -notmatch "Public Transport Monitoring") {
                                throw "Dashboard title not found"
                            }

                            Write-Output "HTML validation passed"
                        '''
                    }
                }

                stage('Check Required Files') {
                    steps {
                        powershell '''
                            foreach ($file in @(
                                "Dockerfile",
                                "deployment.yaml",
                                "Jenkinsfile"
                            )) {
                                if (!(Test-Path $file)) {
                                    throw "$file is missing"
                                }
                            }

                            Write-Output "All required files exist"
                        '''
                    }
                }
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    powershell '''
                        $tag = $env:BUILD_NUMBER
                        $image = "$env:DH_USER/$env:IMAGE"

                        docker build -t "${image}:$tag" .
                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker build failed"
                        }

                        $env:DH_TOKEN |
                            docker login -u $env:DH_USER --password-stdin
                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker login failed"
                        }

                        docker push "${image}:$tag"
                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker push failed"
                        }

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy and Verify') {
            when {
                expression { params.DEPLOY }
            }

            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    ),
                    file(
                        credentialsId: 'docker-desktop-kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    script {
                        try {
                            powershell '''
                                $deployment = kubectl get deployment `
                                    transport-dashboard `
                                    --ignore-not-found -o name

                                if (!$deployment) {
                                    kubectl apply -f deployment.yaml
                                    if ($LASTEXITCODE -ne 0) {
                                        throw "Initial deployment failed"
                                    }
                                }

                                $tag = $env:BUILD_NUMBER

                                if ($env:SIMULATE_BAD_RELEASE -eq "true") {
                                    $tag = "deliberately-missing-image"
                                }

                                $image = "$env:DH_USER/$env:IMAGE`:$tag"

                                kubectl set image `
                                    deployment/transport-dashboard `
                                    "transport-dashboard=$image"

                                if ($LASTEXITCODE -ne 0) {
                                    throw "Image update failed"
                                }

                                kubectl rollout status `
                                    deployment/transport-dashboard `
                                    --timeout=90s

                                if ($LASTEXITCODE -ne 0) {
                                    throw "Rollout failed"
                                }

                                kubectl get deployments
                                kubectl get pods
                                kubectl get services
                            '''
                        } catch (err) {
                            echo "Deployment failed. Attempting rollback."

                            powershell '''
                                kubectl rollout undo `
                                    deployment/transport-dashboard

                                kubectl rollout status `
                                    deployment/transport-dashboard `
                                    --timeout=90s

                                if ($LASTEXITCODE -ne 0) {
                                    throw "Rollback verification failed"
                                }
                            '''

                            error("Deployment failed; rollback was attempted.")
                        }
                    }
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                archiveArtifacts(
                    artifacts: 'index.html,Dockerfile,deployment.yaml,Jenkinsfile',
                    fingerprint: true
                )
            }
        }
    }
}
