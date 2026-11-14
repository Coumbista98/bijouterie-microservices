// Jenkinsfile déclaratif : CI du TP Bijouterie
// Chaîne : Git -> build -> tag -> push Docker Hub -> mise à jour des manifests -> Argo CD
pipeline {
    agent any   // s'exécute sur le Jenkins lui-même (qui a Docker et Git)

    environment {
        DOCKER_USER = 'diagnecoumba'                       // compte Docker Hub
        TAG         = "${env.BUILD_NUMBER}"                // numéro de build = tag (jamais "latest")
        REPO_URL    = 'github.com/Coumbista98/bijouterie-microservices.git'
    }

    stages {

        // Stage 1 : récupération du code depuis le dépôt Git
        // (avec "Pipeline script from SCM", Jenkins fait déjà le clone ; ce stage le rend explicite)
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git config --global --add safe.directory "*"'   // évite l'erreur "dubious ownership"
                sh 'git log -1 --oneline'
            }
        }

        // Stage 2 : construction des trois images à partir de leurs Dockerfiles
        stage('Build') {
            steps {
                sh 'docker build -t bijouterie-frontend ./frontend'
                sh 'docker build -t bijouterie-backend  ./backend'
                sh 'docker build -t bijouterie-mysql    ./database'
            }
        }

        // Stage 3 : chaque image reçoit le nom Docker Hub complet + le numéro de build
        stage('Tag') {
            steps {
                sh 'docker tag bijouterie-frontend $DOCKER_USER/bijouterie-frontend:$TAG'
                sh 'docker tag bijouterie-backend  $DOCKER_USER/bijouterie-backend:$TAG'
                sh 'docker tag bijouterie-mysql    $DOCKER_USER/bijouterie-mysql:$TAG'
            }
        }

        // Stage 4 : connexion à Docker Hub (credential "dockerhub-credentials") puis envoi des images
        stage('Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials',
                                                  usernameVariable: 'HUB_USER',
                                                  passwordVariable: 'HUB_TOKEN')]) {
                    sh 'echo "$HUB_TOKEN" | docker login -u "$HUB_USER" --password-stdin'
                    sh 'docker push $DOCKER_USER/bijouterie-frontend:$TAG'
                    sh 'docker push $DOCKER_USER/bijouterie-backend:$TAG'
                    sh 'docker push $DOCKER_USER/bijouterie-mysql:$TAG'
                }
            }
        }

        // Stage 5 (bonus GitOps) : remplace le tag des images dans les manifests, puis commit + push.
        // Argo CD voit le changement dans Git et déploie la nouvelle version tout seul.
        stage('Update manifests') {
            steps {
                sh '''
                    sed -i "s|image: .*/bijouterie-frontend:.*|image: $DOCKER_USER/bijouterie-frontend:$TAG|" k8s/*-deployment.yaml
                    sed -i "s|image: .*/bijouterie-backend:.*|image: $DOCKER_USER/bijouterie-backend:$TAG|"   k8s/*-deployment.yaml
                    sed -i "s|image: .*/bijouterie-mysql:.*|image: $DOCKER_USER/bijouterie-mysql:$TAG|"       k8s/*-deployment.yaml
                    git config user.email "jenkins@bijouterie.local"
                    git config user.name  "Jenkins CI"
                    git add k8s/
                    git diff --cached --quiet || git commit -m "ci: images en version $TAG [skip ci]"
                '''
                withCredentials([usernamePassword(credentialsId: 'github-credentials',
                                                  usernameVariable: 'GIT_USER',
                                                  passwordVariable: 'GIT_TOKEN')]) {
                    sh 'git push https://$GIT_USER:$GIT_TOKEN@$REPO_URL HEAD:main'
                }
            }
        }
    }

    // Toujours se déconnecter de Docker Hub à la fin
    post {
        always { sh 'docker logout || true' }
    }
}