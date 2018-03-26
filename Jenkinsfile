ansiColor('xterm') {
    properties ([
        buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '10')),
        [$class: 'RebuildSettings', autoRebuild: false, rebuildDisabled: false],
        disableConcurrentBuilds(),
        parameters([choice(choices: '192.168.99.100:8443', description: '', name: 'CLUSTER'),
                     credentials(credentialType: 'org.jenkinsci.plugins.plaincredentials.impl.StringCredentialsImpl', defaultValue: 'openshift-jenkins-slave-text', description: '', name: 'openShiftToken', required: true),
                     booleanParam(defaultValue: true, description: '', name: 'reDeploy'),
                     booleanParam(defaultValue: true, description: '', name: 'deployPrometheus'),
                     booleanParam(defaultValue: true, description: '', name: 'deployKubeStateMetrics'),
                     booleanParam(defaultValue: true, description: '', name: 'deployNodeExporter'),
                     booleanParam(defaultValue: true, description: '', name: 'deployDB'),
                     booleanParam(defaultValue: true, description: '', name: 'deployDBExporter'),
                     booleanParam(defaultValue: true, description: '', name: 'deployAlertmanager'),
                     string(defaultValue: '', description: '', name: 'DB_NAME'),
                     string(defaultValue: '', description: '', name: 'DB_PASSWORD'),
                     string(defaultValue: '', description: '', name: 'ALERTMANAGER_PASSWORD')])

    ])
    def config = [:]
    def nodeLabel = "openshift"
    node(nodeLabel) {
        stage("Connect to Openshift Cluster")
        {
            checkout scm
            def defaultconfig = readProperties file: 'config/DefaultConfiguration'
            // read and print the default config
            echo "*****Contents of defaultconfig map ******"
            // read and print the current openshift's cluster specific config
            config = readProperties defaults: defaultconfig, file: "config/${CLUSTER}"
            echo "*****Contents of cluster specific config******"
            echo "${config}"
            withCredentials([string(credentialsId: '${openShiftToken}', variable: 'tokenid')]) {
                echo "using credentials -> ${openShiftToken}"
                sh "oc login https://${config.CLUSTER_NAME} --token=${tokenid} --insecure-skip-tls-verify"
            }
            sh "oc whoami"
            sh "oc version"
            sh "oc get projects"
            sh "oc project ${config.NAMESPACE}"
        }

        stage("Deploying Prometheus in ${config.NAMESPACE}")
        {
            if(params.deployPrometheus)
            {
                if(params.reDeploy)
                {
                    echo "Removing prometheus"
                    sh "oc delete configmap prometheus-config"
                    sh "oc delete service prometheus"
                    sh "oc delete dc prometheus"
                    sh "oc delete route prometheus"
                }
                def prometheustemplateParameters=[
                    "-p", "NAMESPACE=\"${config.NAMESPACE}\"",
                    "-p", "APPLICATION_NAME=\"${config.APPLICATION_NAME}\"",
                    "-p", "PROMETHEUS_IMAGE=\"${config.PROMETHEUS_IMAGE}\""
                    ].join(" ")
                echo "${prometheustemplateParameters}"
                sh "oc process -f templates/prometheus-template.yaml ${prometheustemplateParameters} | oc apply -f - -n \"${config.NAMESPACE}\""
                sh "oc expose svc/prometheus"
            }
        }

        stage("Deploying Node-Exporter in ${config.NAMESPACE}")
        {
            if(params.deployNodeExporter)
            {
                if(params.reDeploy)
                {
                    echo "Removing Node-Exporter"
                    sh "oc delete service node1"
                    sh "oc delete dc node1"
                    sh "oc delete is node1"
                }
                echo "Deploying Node Exporter"
                sh "oc new-app --name node1 prom/node-exporter"
            }
        }

        stage("Deploying DB in ${config.NAMESPACE}")
        {
           if(params.deployDB)
           {
               if(params.reDeploy)
               {
                   echo "Removing the Database"
                   sh "oc delete service db"
                   sh "oc delete dc db"
               }
               echo "Deploying the Database"
               sh "oc new-app --name db -e MYSQL_ROOT_PASSWORD=${DB_PASSWORD} mariadb"
           }
        }

        stage("Deploying DB-Exporter in ${config.NAMESPACE}")
        {
            if(params.deployDBExporter)
            {
               if(params.reDeploy)
               {
                   echo "Removing the DB-Exporter"
                   sh "oc delete service db-exporter"
                   sh "oc delete dc db-exporter"
                   sh "oc delete is db-exporter"
               }
               echo "Deploying the DB Exporter"
               sh "oc new-app --name db-exporter -e DATA_SOURCE_NAME='root:password@(db:3306)/test' prom/mysqld-exporter" 
            }
        }

        stage("Deploying Kube-State-Metrics in ${config.NAMESPACE}")
        {
            if(params.deployKubeStateMetrics)
            {
               if(params.reDeploy)
               {
                   sh "oc delete service kube-state-metrics"
                   sh "oc delete dc kube-state-metrics"
                   sh "oc  delete is kube-state-metrics"
                   sh "oc delete route kube-state-metrics"
               }
               echo "Deploying Kube-State-Metrics"
               sh "oc new-app --name kube-state-metrics quay.io/coreos/kube-state-metrics:v1.2.0"
            }
        }

        stage("Deploying Alertmanager in ${config.NAMESPACE}")
        {
            if(params.deployAlertmanager)
            {
                if(params.reDeploy)
                {
                    echo "Removing Alertmanager"
                    sh "oc delete service alertmanager"
                    sh "oc delete dc alertmanager"
                    sh "oc delete route alertmanager"
                }
                def alertmanagertemplateParameters=[
                    "-p", "ALERTMANAGER_IMAGE=\"${config.ALERTMANAGER_IMAGE}\"",
                    "-p", "NAMESPACE=\"${config.NAMESPACE}\"",
                    "-p", "ALERTMANAGER_APPLICATION_NAME=\"${config.ALERTMANAGER_APPLICATION_NAME}\"",
                    "-p", "ALERTMANAGER_PASSWORD=\"${ALERTMANAGER_PASSWORD}\""
                    ].join(" ")
                sh "oc process -f templates/alertmanager-template.yml ${alertmanagertemplateParameters} | oc apply -f - -n \"${config.NAMESPACE}\""
                sh "oc expose svc/alertmanager"
            }
        }
    }
}