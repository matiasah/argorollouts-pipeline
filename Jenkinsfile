/*
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-slave
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-slave-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins-slave
  namespace: jenkins
*/

pipeline {

    parameters {
        choice(description: "Action", name: "Action", choices: ["Plan", "Apply", "Destroy"])
    }

    agent {
        kubernetes {
            yaml """
                apiVersion: "v1"
                kind: "Pod"
                spec:
                  securityContext:
                    runAsUser: 1001
                    runAsGroup: 1001
                    fsGroup: 1001
                  containers:
                  - command:
                    - "cat"
                    image: "alpine/helm:latest"
                    imagePullPolicy: "IfNotPresent"
                    name: "helm"
                    resources: {}
                    tty: true
                    volumeMounts:
                    - mountPath: "/.config"
                      name: "config-volume"
                      readOnly: false
                    - mountPath: "/.cache/helm/"
                      name: "cache-volume"
                      readOnly: false
                  - command:
                    - "cat"
                    image: "bitnami/kubectl:latest"
                    imagePullPolicy: "IfNotPresent"
                    name: "kubectl"
                    resources: {}
                    tty: true
                  - command:
                    - "cat"
                    image: "k8s.gcr.io/kustomize/kustomize:v5.0.1"
                    imagePullPolicy: "IfNotPresent"
                    name: "kustomize"
                    resources: {}
                    tty: true
                  serviceAccountName: jenkins-slave
                  volumes:
                  - emptyDir:
                      medium: ""
                    name: "config-volume"
                  - emptyDir:
                      medium: ""
                    name: "cache-volume"
            """
        }
    }

    stages {

        stage ("Helm Repo") {

            steps {

                container ("helm") {

                    script {
    
                        // Install repo
                        sh "helm repo add argo https://argoproj.github.io/argo-helm"
                        sh "helm repo update"
    
                    }

                }

            }

        }

        stage ("Namespace: Apply") {

            when {

                expression {
                    return env.ACTION.equals("Apply")
                }

            }

            steps {

                container ("kubectl") {

                    script {

                        // Apply
                        sh "kubectl create namespace argo-rollouts --dry-run=client -o yaml | kubectl apply -f -"

                    }

                }

            }

        }

        stage ("Template: Plan") {

            steps {

                container ("helm") {

                    script {

                        // Cluster agent options
                        OPTIONS = " "

                        // Template
                        sh "helm template argo-rollouts -f argo-rollouts-values.yaml argo/argo-rollouts ${OPTIONS} --namespace argo-rollouts > argo-rollouts-base.yaml"
        
                    }

                }

                container ("kustomize") {
  
                    script {
  
                        // Kustomize
                        sh "kustomize build > argo-rollouts.yaml"

                        // Print Yaml
                        sh "cat argo-rollouts.yaml"
  
                    }
  
                }

            }

        }

        stage ("Template: Apply") {

            when {
                
                expression {
                    return !env.ACTION.equals("Plan")
                }
                
            }

            steps {

                container ("kubectl") {

                    script {

                        // Apply
                        if (env.ACTION.equals("Apply")) {

                            // Apply
                            sh "kubectl apply -f argo-rollouts.yaml"

                        // Destroy
                        } else if (env.ACTION.equals("Destroy")) {

                            try {
                                
                                // Destroy
                                sh "kubectl delete -f argo-rollouts.yaml"

                            } catch (Exception e) {

                                // Do nothing

                            }

                        }

                        sh "rm argo-rollouts.yaml"

                    }

                }

            }

        }

        stage ("Restart") {

            when {
                
                expression {
                    return !env.ACTION.equals("Plan")
                }
                
            }

            steps {

                container ("kubectl") {

                    script {

                        sh "kubectl rollout restart deployment -n argo-rollouts"
                        sh "kubectl rollout restart daemonset -n argo-rollouts"

                    }

                }

            }

        }

    }
    
}
