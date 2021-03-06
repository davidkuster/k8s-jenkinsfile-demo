#!/usr/bin/env groovy
@Library([
    'github.com/fabric8io/fabric8-pipeline-library@master',
    'github.com/davidkuster/jenkins-pipeline-library@master'
])

def pipelineUtils = new net.talldave.pipeline.PipelineUtils()
def containerUtils = new net.talldave.pipeline.ContainerUtils()

// apply k8s files to the given namespace
void kubernetesDeploy(String namespace) {
    echo "Deploying to namespace $namespace..."

    // load Fabric8 files
    def kubefiles = findFiles(glob: '**target/classes/META-INF/fabric8/kubernetes/*.yml')
    echo "Found # of kubernetes.yml files: ${kubefiles.length}"

    for (i = 0; i < kubefiles.length; i++) {
        def file = kubefiles[i]
        echo "Deploying: ${file.path}"
        def fileContent = readFile(file.path).replace('((NAMESPACE))', namespace)

        // deploy this to k8s, and wait up to 20 minutes for readiness checks to pass
        kubernetesApply(file: fileContent, environment: namespace) //, registry: 'custom...')
        // TODO: use "readinessTimeout: 1200000" once v1.6 of kubernetes-pipeline-devops-steps and kubernetes-pipeline-steps is released
    }
}

// run verify script once the deployment has successfully rolled out
void verify(String namespace) {
    sleep 10

    container('k8s') {
        def status = sh(returnStdout: true, script: "kubectl --namespace=$namespace rollout status deployment/demo -w")
        echo "rollout status: \n$status"
        if (! status.contains("successfully rolled out")) {
            error "Kubernetes deploy failed"
        }
    }

    // can include automated testing against the new deployment
}


// typically read from Jenkins job parameters or somewhere instead of hard coding...
def namespaces = [ 'dev', 'qa', 'production' ]
// deploy to the first namespace automatically, then prompt the user for each subsequent namespace


String label = pipelineUtils.getNodeLabel()

podTemplate(
    cloud: 'kubernetes',
    label: label,
    containers: [
        containerUtils.mavenContainer(),
        containerUtils.kubectlContainer(name: 'k8s')
    ],
    volumes: [
        hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')
    ],
    envVars: [
        podEnvVar(key: 'DOCKER_HOST', value: 'unix:/var/run/docker.sock'),
        podEnvVar(key: 'DOCKER_CONFIG', value: '/home/jenkins/.docker/')
    ]
) {
    node(label) {

        echo "checking out source..."
        checkout scm

        String version = pipelineUtils.getBuildVersion()

        stage('Build Project') {
            container('maven') {
                // set to a more meaningful/custom value than what's in the pom
                sh "mvn org.codehaus.mojo:versions-maven-plugin:2.5:set -U -DnewVersion=$version"
                sh 'mvn clean package'
            }
        }

        stage('Build and Push Docker Image') {
            container('maven') {
                // rebuild k8s resource descriptors here so all timestamps are in sync
                sh 'mvn fabric8:resource fabric8:build' // fabric8:push'
            }
        }

        // iterate over the parameterized namespaces
        for (int x=0; x < namespaces.size(); x++) {

            String namespace = namespaces[x]

            echo "processing namespace $namespace"

            // deploy to the first namespace automatically. Require a manual gate after the first namespace.
            if (x > 0) {
                stage("Approve $namespace") {
                    timeout(time: 7, unit: 'DAYS') {
                        input(message: "Would you like to deploy this build to the $namespace namespace?")

                        // Note that because this is inside a "node" block it keeps the agent pod running while
                        // potentially waiting for manual approval to happen.

                        // It's possible to move user input outside of a "node" block but every time a node block ends
                        // it shuts down that agent pod, and a new node block starts a new agent pod.

                        // So, if we make all deploy/verify steps happen in a new node block it will slow the
                        // pipeline down as it's going to be spinning agent pods up and down a lot.

                        // Also need to use stash/unstash in that scenario so the artifacts from one agent pod can be
                        // available to subsequent agent pods.
                    }
                }
            }

            stage("Deploy $namespace") {
                kubernetesDeploy(namespace)
            }

            stage("Verify $namespace") {
                verify(namespace)
            }
        }
    }
}
