pipeline {
    agent any
    //设置Gitlab事件触发Jenkins流水线
    triggers {
        gitlab(triggerOnPush: ture,
        triggerOnMergeRequest: ture,
        branchFilterType: 'All',
        secretToken: "7fe72e5b8dc2f54ceaa25bb9bb986da7")
    }
    //设置环境变量"保留变量名--environment"，其中credentials()为调用jenkins预设凭证
    environment {
        HARBOR = credentials('HARBOR_USER')
        KUBERNETES = credentials('K8S_CONF')
        GIT_TAG = sh(returnStdout: true,script: 'git describe --tags --always').trim()
    }
    //设置参数化流水线"保留变量名--parameters"，使用string()用于传入字符串参数。
    parameters {
        string(name: 'HARBOR_HOST', defaultValue: '172.16.1.27', description: 'harbor仓库地址')
        string(name: 'DOCKER_IMAGE', defaultValue: 'project-01/demo01', description: 'docker镜像名')
        string(name: 'APP_NAME', defaultValue: 'demo01', description: 'k8s中标签名')
        string(name: 'K8S_NAMESPACE', defaultValue: 'demo', description: 'k8s的namespace名称')
    }
    //开始进行流水线配置
    stages {
        stage('Code Build') {
            agent { 
                docker {
                    image 'maven:3-alpine'
                    args '-v $HOME/.m2:/root/.m2'
                    } 
            }
            steps {
                echo "开始用mave构建Java代码"
                sh 'mvn --version'
                //使用stash关键字暂时将文件存储起来（官方没有查询到文件具体存储地址）
                stash name: 'app', includes: 'target/*.jar'
            }
        }
        stage('Create Docker') {
            agent {
                docker {
                    image 'docker:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                echo "开始创建镜像并上传到镜像仓库"
                //使用unstash关键字调用stash暂存的文件，其中调用的名字和stash name的名字保持一致
                unstash 'app'
                sh "docker login -u ${USERNM} -p ${PASSWD} ${params.HARBOR_HOST}"
                sh "docker build --build-arg JAR_FILE=`ls target/*.jar |cut -d '/' -f2` -t ${params.HARBOR_HOST}/${params.DOCKER_IMAGE}:${GIT_TAG} ."
                sh "docker push ${params.HARBOR_HOST}/${params.DOCKER_IMAGE}:${GIT_TAG}"
                sh "docker rmi ${params.HARBOR_HOST}/${params.DOCKER_IMAGE}:${GIT_TAG}"
            }
        }
        stage('Deploy Kubernetes') {
            agent {
                docker {
                    image 'kubectl:latest'
                }
            }
            step {
                echo "开始在Rancher平台上部署容器应用"
                sh "mkdir -p ~/.kube"
                //通过"K8S_CONF"调用jenkins里面预设的kubernetes config文件到容器内部
                sh "echo ${K8S_CONF} | base64 -d > ~/.kube/config"
                //替换实际环境的配置参数到.tmpl模版文件，然后将tmpl模版文件重写进kubernetes部署的yaml文件用于Rancher平台部署
                sh "sed -e 's#{IMAGE_URL}#${params.HARBOR_HOST}/${params.DOCKER_IMAGE}#g;s#{IMAGE_TAG}#${GIT_TAG}#g;s#{APP_NAME}#${params.APP_NAME}#g;s#{SPRING_PROFILE}#k8s-test#g' k8s-deployment.tmpl > k8s-deployment.yml"
                sh "kubectl apply -f k8s-deployment.yml --namespace=${params.K8S_NAMESPACE}"
            }
        }
    }
}
