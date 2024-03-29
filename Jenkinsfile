pipeline {

    agent any

    environment {
        DB_USER=credentials('psql-user')
        DB_PASS=credentials('psql-pass')
        DB_NAME=credentials('psql-db')
        
    }

    stages {

        stage('Build') {
            steps {
                // Get some code from a GitHub repository
                git branch: 'main', url: 'https://github.com/panagiotisbellias/e-movies-app.git'
            }
        }
        
        stage('Test') {
            steps {
                sh '''
                    python3 -m venv myvenv
                    source myvenv/bin/activate
                    pip install -r requirements.txt

                    cd movies_app
                    cp movies_app/.env.example movies_app/.env
                    ./manage.py test
                '''
            }
        }

        stage('Ansible Deployment') {

            environment {
                DB_URL=credentials('ansible-db-url')
                HOSTS=credentials('ansible-hosts')
                SECRET_KEY=credentials('django-key')
            }

            steps {
                sshagent (credentials: ['ssh-ansible-vm']) {

                sh '''
                    cd ~/workspace/ansible-movie-code
                    ansible-playbook -l gcloud_ansible playbooks/postgres-install.yml \
                    -e PSQL_USER=$DB_USER \
                    -e PSQL_PASSWD=$DB_PASS \
                    -e PSQL_DB=$DB_NAME

                    ansible-playbook -l gcloud_ansible playbooks/django-install.yml \
                    -e SECRET_KEY=$SECRET_KEY \
                    -e DATABASE_URL=$DB_URL \
                    -e ALLOWED_HOSTS=$HOSTS
                    '''
                }
            }
        }
        
        stage('Docker Deployment') {

            environment {
                DB_URL=credentials('docker-db-url')
                HOSTS=credentials('docker-hosts')
                SECRET_KEY=credentials('django-key')
            }

            steps {
                sshagent (credentials: ['ssh-docker-vm']) {
                    sh '''
                        cd ~/workspace/ansible-movie-code
                        ansible-playbook -l gcloud_docker playbooks/django-docker.yml \
                        -e SECRET_KEY=$SECRET_KEY \
                        -e DATABASE_URL=$DB_URL \
                        -e ALLOWED_HOSTS=$HOSTS
                    '''
                }
            }
        }

        stage('Preparing k8s Deployment') {

            environment {
                IMAGE=credentials('docker-image')
                DOCKER_USERNAME=credentials('docker-user')
                DOCKER_PASSWORD=credentials('docker-pass')
            }
            
            steps {
                sh '''
                    echo $BUILD_ID
                    COMMIT_ID=$(git rev-parse --short HEAD)
                    echo $COMMIT_ID
                    TAG=$COMMIT_ID-$BUILD_ID
                    docker build -t $IMAGE -t $IMAGE:$TAG -f nonroot.Dockerfile .
                    docker login -u="$DOCKER_USERNAME" -p="$DOCKER_PASSWORD"
                    docker push $IMAGE --all-tags
                '''
            }
        }

        stage('Kubernetes Deployment') {
            
            environment {
                DB_URL=credentials('k8s-db-url')
                HOSTS=credentials('k8s-hosts')
                SECRET_KEY=credentials('django-key')
            }

            steps {
                sh '''
                    kubectl config use-context microk8s

                    kubectl create secret generic pg-user \
                    --from-literal=PGUSER=$DB_USER \
                    --from-literal=PGPASSWORD=$DB_PASS \
                    --from-literal=PGDATABASE=$DB_NAME --dry-run -o yaml \
                    | kubectl apply -f -
                    
                    cd ~/workspace/ansible-movie-code
                    ansible-playbook playbooks/django-populate-env.yml \
                    -e SECRET_KEY=$SECRET_KEY \
                    -e DATABASE_URL=$DB_URL \
                    -e ALLOWED_HOSTS=$HOSTS
                    
                    cd ~/workspace/e-movies-app
                    kubectl create configmap django-config \
                    --from-env-file=movies_app/movies_app/.env \
                    --dry-run -o yaml | kubectl apply -f -

                    cd k8s
                    kubectl apply -f db/postgres-pvc.yaml
                    kubectl apply -f db/postgres-deployment.yaml
                    kubectl apply -f db/postgres-clip.yaml
                    kubectl apply -f django/django-deployment.yaml
                    kubectl apply -f django/django-clip.yaml
                    kubectl apply -f django/django-ingress.yaml
                    kubectl apply -f django/django-https-ingress.yaml
                '''
            }
        }

    }
}