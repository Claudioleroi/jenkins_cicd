# Jenkins CI/CD Automatisé sur EKS Kubernetes

## Une exploration des opérateurs Kubernetes, de la pipeline Jenkins en tant que code, et de la surveillance avec Grafana, Loki et Prometheus

### Prérequis :
- Compte AWS Cloud [AWS](https://aws.amazon.com/)
- Un utilisateur IAM avec une clé d'accès et une clé secrète ayant une politique d'administrateur
- AWS CLI connecté, installez-le via [AWS CLI Installation Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) et configurez les clés avec la commande `aws configure`
- Outil `eksctl` : installez Kubernetes depuis [eksctl GitHub](https://github.com/eksctl-io/eksctl/releases/tag/v0.161.0)
- Installez `kubectl` pour vous connecter à Kubernetes [kubectl Installation Guide](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)
- Installez `helm` pour installer les opérateurs [helm GitHub](https://github.com/helm/helm/releases)

### Processus étape par étape :

1. **Créer un Cluster/OS/Container avec la commande `kubectl`**
    ```sh
    eksctl create cluster --name mycluster --region ap-southeast-1
    ```
    Ce processus prend environ 10-15 minutes. Après la création du cluster, vérifiez son état avec la commande suivante.

2. **Créer un namespace**
    ```sh
    kubectl create namespace myns
    ```

3. **Installer l'opérateur Jenkins avec `helm`**
    - Ajouter l'opérateur Jenkins à notre référentiel :
      ```sh
      ./helm.exe repo add jenkins https://raw.githubusercontent.com/jenkinsci/kubernetes-operator/master/chart
      ```
    - Installer l'opérateur Jenkins depuis notre référentiel :
      ```sh
      ./helm.exe install my-jenkins-operator jenkins/jenkins-operator -n myns --set jenkins.enabled=false
      ```
      Note : Si la deuxième commande ne fonctionne pas bien, cela peut être dû à un réseau non pris en charge. Veuillez activer un VPN (comme Proton) pour accéder au code brut.

4. **Créer des jobs pour la pipeline Jenkins**
    Créez un fichier `jenkins_instance.yml` contenant le code suivant :
    ```yaml
    apiVersion: jenkins.io/v1alpha2
    kind: Jenkins
    metadata:
      name: example
      namespace: lwns
    spec:
      configurationAsCode:
        configurations: []
        secret:
          name: ""
      groovyScripts:
        configurations: []
        secret:
          name: ""
      jenkinsAPISettings:
        authorizationStrategy: createUser
      master:
        disableCSRFProtection: false
        containers:
          - name: jenkins-master
            image: jenkins/jenkins:lts
            imagePullPolicy: Always
            livenessProbe:
              failureThreshold: 12
              httpGet:
                path: /login
                port: http
                scheme: HTTP
              initialDelaySeconds: 100
              periodSeconds: 10
              successThreshold: 1
              timeoutSeconds: 5
            readinessProbe:
              failureThreshold: 10
              httpGet:
                path: /login
                port: http
                scheme: HTTP
              initialDelaySeconds: 80
              periodSeconds: 10
              successThreshold: 1
              timeoutSeconds: 1
            resources:
              limits:
                cpu: 1500m
                memory: 3Gi
              requests:
                cpu: "1"
                memory: 500Mi
      seedJobs:
        - id: jenkins-operator
          targets: "cicd/jobs/*.jenkins"
          description: "LW Jenkins Operator repository"
          repositoryBranch: master
          repositoryUrl: https://github.com/jenkinsci/kubernetes-operator.git
    ```

    Exécutez le fichier `yml` :
    ```sh
    kubectl create -f jenkins_instance.yaml
    ```
    Vérifiez le statut de la création des jobs :
    ```sh
    kubectl --namespace lwns get pods -w
    ```

5. **Accéder à l'interface Jenkins**
    - Pour obtenir l'URL, le nom d'utilisateur et le mot de passe, utilisez les commandes ci-dessous :
    ```sh
    kubectl --namespace lwns get secret jenkins-operator-credentials -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode
    ```

6. **Surveillance avec Grafana, Loki et Prometheus**
    Installez les logiciels avec la commande suivante :
    ```sh
    ./helm.exe upgrade --install loki grafana/loki-stack --set grafana.enabled=true,prometheus.enabled=true,prometheus.alertmanager.persistentVolume.enabled=false,prometheus.server.persistentVolume.enabled=false
    ```
    Importez les IDs pour les tableaux de bord Loki et Prometheus dans l'interface Grafana.

7. **Nettoyer les ressources**
    Supprimez les ressources créées :
    ```sh
    ./helm delete loki
    kubectl delete namespace myns
    ```

### Conclusion
En résumé, notre projet utilise la puissance de EKS Kubernetes pour déployer et gérer facilement Jenkins CI/CD grâce à l'automatisation. Grâce à des outils innovants, nous avons simplifié la création de pipelines Jenkins en tant que code, le lancement de jobs d'application sur des nœuds provisionnés dynamiquement, et la surveillance de notre infrastructure avec Grafana, Loki et Prometheus.

Merci de nous avoir accompagnés dans ce voyage vers un développement logiciel et une gestion d'infrastructure plus efficaces.
