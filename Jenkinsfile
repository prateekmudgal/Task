pipeline {
    
    agent any
    environment{
        SONAR_HOME = tool "Sonar"
    }
    stages {
        
        stage("Code"){
            steps{
                git url: "https://github.com/prateekmudgal/DevSecOps_ToDo_app_Jenkins_sonarqube.git" , branch: "main"
                echo "Code Cloned Successfully"
            }
        }
        stage("SonarQube Analysis"){
            steps{
               withSonarQubeEnv("Sonar"){
                   sh "${SONAR_HOME}/bin/sonar-scanner -Dsonar.projectName=ToDo_app -Dsonar.projectKey=ToDo_app -X"
               }
            }
        }
        stage("SonarQube Quality Gates"){
            steps{
               timeout(time: 5, unit: "MINUTES"){
                   waitForQualityGate abortPipeline: false
               }
            }
        }
        
        stage("Build & Test"){
            steps{
                sh 'docker build -t prateek0912/todo_app:latest .'
                echo "Code Built Successfully"
            }
        }
        stage("Trivy"){
            steps{
                sh "trivy image prateek0912/todo_app:latest"
            }
        }
        stage("Push to Private Docker Hub Repo"){
            steps{
                withCredentials([usernamePassword(credentialsId: 'DockerHub', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USERNAME')]) {
                sh "docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASS}"
                sh "docker push prateek0912/todo_app:latest"
                }
                
            }
        }
        stage("Deploy"){
            steps{
                sh "docker-compose up -d"
                echo "App Deployed Successfully"
            }
        }
    }
}