pipeline{
    agent { label "dev"};
    
    stages{
        stage("CODE"){
            steps{
                script{
                   clone("https://github.com/pratikk-devops/two-tier-flask-app.git", "main")
               }                
            }
        }
        stage("Trivy File System Scan"){
            steps{
                script{
                    trivy_fs()
                }
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
