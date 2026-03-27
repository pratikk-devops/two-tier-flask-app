pipeline{
    agent any;
    
    stages{
        stage("CODE"){
            steps{
                    git url: "https://github.com/pratikk-devops/two-tier-flask-app.git", branch: "main"            
            }
        }
        stage("BUILD"){
            steps{
                sh "docker build -t two-tier-flask-app ."
            }
        }
        stage("TEST")  {
            steps{
                echo"Developer/Tester writes the tests"
            }
        }
        stage("Push to Docker Hub")  {
            steps{
                withCredentials([usernamePassword(
                    credentialsId:"dockerHubCreds", 
                    passwordVariable: "dockerHubPass",
                    usernameVariable: "dockerHubUser"
               )]){
                sh "docker login -u ${env.dockerHubUser} -p ${env.dockerHubPass} "
                sh "docker image tag two-tier-flask-app ${env.dockerHubUser}/two-tier-flask-app"
                sh "docker push ${env.dockerHubUser}/two-tier-flask-app:latest"
                }
            }
        }
        stage("DEPLOY"){
            steps{
                sh "docker compose up -d --build flask-app"
            }
        }
    }
}
