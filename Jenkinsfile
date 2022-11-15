node {
    def app

    stage('Clone repository') {
        /* Let's make sure we have the repository cloned to our workspace */

        checkout scm
    }

    stage('Build image') {
        /* This builds the actual image; synonymous to
         * docker build on the command line */

        app = docker.build("bajacloud/nodeapp")
    }

    stage('Scan image with twistcli') {
        withCredentials([usernamePassword(credentialsId: 'twistlock_creds', passwordVariable: 'TL_PASS', usernameVariable: 'TL_USER')]) {
            sh 'curl -k -u $TL_USER:$TL_PASS --output ./twistcli https://$TL_CONSOLE/api/v1/util/twistcli'
            sh 'sudo chmod a+x ./twistcli'
            sh "./twistcli images scan --u $TL_USER --p $TL_PASS --address https://$TL_CONSOLE --details bajacloud/nodeapp:latest"
        }
    }
    stage('Test image') {
        /* Ideally, we would run a test framework against our image.
         * For this example, we're using a Volkswagen-type approach ;-) */

        app.inside {
            sh 'echo "Tests passed"'
        }
    }

    stage('Push image') {
        /* Finally, we'll push the image with two tags:
         * First, the incremental build number from Jenkins
         * Second, the 'latest' tag.
         * Pushing multiple tags is cheap, as all the layers are reused. */
        docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
            app.push("${env.BUILD_NUMBER}")
            app.push("latest")
        }
    }

    stage('Get Jenkins scans') {
        withCredentials([usernamePassword(credentialsId: 'twistlock_creds', passwordVariable: 'TL_PASS', usernameVariable: 'TL_USER')]) {
            sh 'curl -s -k -u $TL_USER:$TL_PASS https://$TL_CONSOLE/api/v1/scans?type=jenkins |  jq --raw-output \'.[] | select(.pass==true) | [.jobName, .build, .info.id] | @csv\' | tr -d \\\" > jenkins_ids.csv'
        }
    }
    
    stage('Push Trusted images') {
        withCredentials([usernamePassword(credentialsId: 'twistlock_creds', passwordVariable: 'TL_PASS', usernameVariable: 'TL_USER')]) {
            sh '''        while IFS=, read -r "jobName" "build" "id"; do
                curl -s -k -u $TL_USER:$TL_PASS https://$TL_CONSOLE/api/v1/trust -X POST -H "Content-Type:application/json" -v -d "{\\\"image\\\":\\\"${id}\\\", \\\"_id\\\":\\\"Jenkins: ${jobName} Build ${build} \\\"}"
                done <jenkins_ids.csv
               '''
        }
}
}

