

---

# DevSecOps ToDo Application

This project demonstrates the creation of a **Node.js ToDo application** with a DevSecOps approach. We leverage Jenkins for CI/CD, SonarQube for code quality and security analysis, and Trivy for Docker image scanning.

## Running the Project

To execute the pipeline:

1. Clone the repository:
    ```bash
    git clone https://github.com/prateekmudgal/DevSecOps_ToDo_app_Jenkins_sonarqube.git
    ```
2. Open Jenkins and create a new pipeline.
3. Link it to this repository and select the `Jenkinsfile` for the pipeline script.
4. Run the pipeline and monitor the stages.

## Tools and Technologies

To build and deploy this application, the following tools are required:

1. **AWS EC2**: To host the Jenkins server and other necessary services.
2. **Docker & Docker-Compose**: For containerization of the application.
3. **DockerHub**: To store and retrieve Docker images.
4. **GitHub**: Version control and repository hosting.
5. **Jenkins**: For Continuous Integration and Continuous Deployment (CI/CD).
6. **SonarQube**: For static code analysis and security scanning.
7. **Trivy**: For scanning Docker images for vulnerabilities.

## Prerequisites

Ensure you have the following set up before proceeding:

- An AWS EC2 instance
- Docker and Docker-Compose installed on the instance
- A GitHub account 
- Jenkins installed on the EC2 instance
- SonarQube and Trivy installed on the EC2 instance

## Project Setup

### Step 1: Launch EC2 Instance

Start by launching an EC2 instance on AWS, which will serve as your server for Jenkins, Docker, SonarQube, and Trivy.

### Step 2: Install Jenkins

1. Update your package lists:
    ```bash
    sudo apt upgrade && sudo apt upgrade -y
    ```
2. Install Java (Jenkins requires Java):
    ```bash
    sudo apt install fontconfig openjdk-17-jre
    ```
3. Check the Java version:
    ```bash
    java --version
    ```
4. Install Jenkins:
    ```bash
    sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
    echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
    sudo apt-get update
    sudo apt-get install jenkins
    sudo systemctl start jenkins
    sudo systemctl enable jenkins
    ```
5. Check Jenkins status:
    ```bash
    sudo service jenkins status
    ```

### Step 3: Install Docker and Docker-Compose

1. Install Docker and Docker-Compose:
    ```bash
    sudo apt-get update
    sudo apt-get install docker.io docker-compose -y
    ```
2. Add your user and Jenkins to the Docker group:
    ```bash
    sudo usermod -aG docker $USER
    sudo usermod -aG docker jenkins
    sudo reboot
    ```
3. Verify Docker installation:
    ```bash
    docker version
    ```
4. Enable Docker:
    ```bash
    sudo systemctl enable docker
    ```

### Step 4: Setup SonarQube

1. Run SonarQube in a Docker container:
    ```bash
    docker run -itd --name sonarqube -p 9000:9000 sonarqube:lts-community
    ```
To make the DevSecOps pipeline, you first need to create a user on SonarQube who will have access provided to Jenkins. Start by navigating to **SonarQube** and going to **Administrator > Security > Users > Tokens**. Update the tokens by naming one "jenkins" and generating it. With the SonarQube setup done, you can proceed to configure Jenkins to work with SonarQube.

Begin by installing the necessary SonarQube plugins in Jenkins. Go to **Manage Jenkins**, then **Plugins**, and search for "SonarQube Scanner" under **Available Plugins**. Install this plugin.

Next, you need to add the SonarQube token to Jenkins. Go to **Manage Jenkins**, click on **Credentials**, then **System**, and select **Global credentials**. Add a new credential as **Secret text**, where you will paste the token copied from SonarQube. Name this credential "Sonar" and provide a description, then create it.

Similarly, you should add Docker credentials in Jenkins. Navigate to **Manage Jenkins**, then **Credentials**, and again to **System** under **Global credentials**. Add a new credential as **Username with password**. Enter your DockerHub username and password, name it "DockerHub," provide a description, and create it.

After setting up the credentials, you will link SonarQube with Jenkins. Go to **Manage Jenkins**, then **System**, and find **SonarQube servers**. Add a new SonarQube server with the name "Sonar" and set the **Server URL** to `http://IP:9000`. Use the previously created Sonar authentication token for the server, then apply and save these settings.

To enable Sonar Scanner, return to **Manage Jenkins**, then **Tools**, and find **SonarQube Scanner installations**. Add a new installation named "Sonar," set the version to "latest," and apply and save the configuration.

Lastly, create Sonar webhooks by going to **SonarQube**, then **Administrator**, and selecting **Webhooks**. Create a new webhook named "jenkins" with the URL set to `http://IP:8080/sonarqube-webhook/`, and then create it.

### Step 5: Install Trivy

1. Install Trivy:
    ```bash
    sudo apt-get install wget apt-transport-https gnupg lsb-release
    wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
    echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
    sudo apt-get update
    sudo apt-get install trivy
    ```

## Jenkins CI/CD Pipeline

The pipeline for this project is defined in the `Jenkinsfile` and includes the following stages:

1. **Code**: Clones the repository.
2. **SonarQube Analysis**: Analyzes the code using SonarQube.
3. **SonarQube Quality Gates**: Waits for quality gate results.
4. **Build & Test**: Builds the Docker image for the application.
5. **Trivy**: Scans the Docker image for vulnerabilities using Trivy.
6. **Push to Private Docker Hub Repo**: Pushes the Docker image to DockerHub.
7. **Deploy**: Deploys the application using Docker-Compose.

### Pipeline Script

```groovy
pipeline {
    agent any
    environment {
        SONAR_HOME = tool "Sonar"
    }
    stages {
        stage("Code") {
            steps {
                git url: "https://github.com/prateekmudgal/DevSecOps_ToDo_app_Jenkins_sonarqube.git", branch: "main"
                echo "Code Cloned Successfully"
            }
        }
        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv("Sonar") {
                    sh "${SONAR_HOME}/bin/sonar-scanner -Dsonar.projectName=ToDo_app -Dsonar.projectKey=ToDo_app -X"
                }
            }
        }
        stage("SonarQube Quality Gates") {
            steps {
                timeout(time: 5, unit: "MINUTES") {
                    waitForQualityGate abortPipeline: false
                }
            }
        }
        stage("Build & Test") {
            steps {
                sh 'docker build -t prateek0912/todo_app:latest .'
                echo "Code Built Successfully"
            }
        }
        stage("Trivy") {
            steps {
                sh "trivy image prateek0912/todo_app:latest"
            }
        }
        stage("Push to Private Docker Hub Repo") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'DockerHub', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USERNAME')]) {
                    sh "docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASS}"
                    sh "docker push prateek0912/todo_app:latest"
                }
            }
        }
        stage("Deploy") {
            steps {
                sh "docker-compose up -d"
                echo "App Deployed Successfully"
            }
        }
    }
}
```

## Application Output

After successfully deploying, you can view the ToDo application. Here’s a preview:



---

# Thank You

I hope you find it useful. If you have any doubt in any of the step then feel free to contact me. If you find any issue in it then let me know.

<table>
  <tr>
    <th><a href="https://www.linkedin.com/in/prateek-mudgal-devops" target="_blank"><img src="https://img.icons8.com/color/452/linkedin.png" alt="linkedin" width="30"/></a></th>
    <th><a href="mailto:mudgalprateek00@gmail.com" target="_blank"><img src="https://img.icons8.com/color/344/gmail-new.png" alt="Mail" width="30"/></a></th>
  </tr>
</table>
