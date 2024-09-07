pipeline {
    agent {
        kubernetes {
            label 'maven'
        }
    }

    // parameters {
    //     gitParameter name: 'BRANCH_NAME', branch: '', branchFilter: '.*', defaultValue: 'origin/master', description: '请选择要发布的分支', quickFilterEnabled: false, selectedValue: 'NONE', tagFilter: '*', type: 'PT_BRANCH'
    //     choice(name: 'NAMESPACE', choices: ['ns-dev', 'ns-test', 'ns-test', 'ns-prod'], description: '命名空间')
    //     string(name: 'TAG_NAME', defaultValue: 'snapshot', description: '标签名称，必须以 v 开头，例如：v1、v1.0.0')
    // }

    environment {
        DOCKER_CREDENTIAL_ID = 'harbor-user-pass'
        GIT_REPO_URL = '192.168.0.134'
        GIT_CREDENTIAL_ID = 'git-user-pass'
        GIT_ACCOUNT = 'root' // change me
        KUBECONFIG_CREDENTIAL_ID = '8215a3bb-0576-4e5c-9a51-d734b0835047'
        REGISTRY = '192.168.0.200:30002'
        DOCKERHUB_NAMESPACE = 'snapshots' // change me
        APP_NAME = 'k8s-cicd-demo'
        SONAR_SERVER_URL = 'http://192.168.0.200:32603/'
        SONAR_CREDENTIAL_ID = 'sonarqube-token'
        DOCKER_IMAGE = "${REGISTRY}/${DOCKERHUB_NAMESPACE}/${APP_NAME}:SNAPSHOT-${BUILD_NUMBER}"
        PROJECT_URL = "http://192.168.0.134/root/k8s-cicd-demo.git"
        PROJECT_NAMESPACE = "ns-test"
        BRANCH_NAME = "origin/test"
        DOCKER_PATH = "./Dockerfile"
        VALUESFILE_PATH = "./$APP_NAME/values-test.yaml"
    }

    stages {

        stage('checkout scm') {
            steps {
                checkout scmGit(branches: [[name: "$BRANCH_NAME"]], extensions: [], userRemoteConfigs: [[credentialsId: "$GIT_CREDENTIAL_ID", url: "$PROJECT_URL"]])
            }
        }

        stage('unit test') {
            steps {
                sh 'mvn clean test'
            }
        }

        stage('build & push') {
            steps {
                sh 'mvn clean package -DskipTests'
                sh 'docker build -f Dockerfile -t $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER .'
                withCredentials([usernamePassword(passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME', credentialsId: "$DOCKER_CREDENTIAL_ID",)]) {
                    sh 'echo "$DOCKER_PASSWORD" | docker login $REGISTRY -u "$DOCKER_USERNAME" --password-stdin'
                    sh 'docker push $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER'
                }
            }
        }

        stage('pull charts') {
            steps {
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'git-user-pass', url: 'http://192.168.0.134/devops/helm-charts.git']])
            }
        }

        stage('deploy to dev') {

            steps {
                input(id: 'deploy-to-dev', message: 'deploy to dev?')
                sh '''
                    helm upgrade $APP_NAME ./$APP_NAME  -f $VALUESFILE_PATH --namespace=$PROJECT_NAMESPACE  --install --set image.repository=$DOCKER_IMAGE --atomic --timeout 5m --cleanup-on-fail --create-namespace --force --debug
                '''
            }
        }
    }
}