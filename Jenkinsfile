docker.image('cloudbees/java-build-tools:0.0.5').inside {

    checkout scm
    def mavenSettingsFile = "${pwd()}/.m2/settings.xml"

    stage 'Build Web App'
    wrap([$class: 'ConfigFileBuildWrapper',
        managedFiles: [[fileId: 'maven-settings-for-gameoflife', targetLocation: "${mavenSettingsFile}"]]]) {

        sh "mvn -s ${mavenSettingsFile} -DaltSnapshotDeploymentRepository=cleclerc.artifactoryonline.com::default::https://cleclerc.artifactoryonline.com/cleclerc/libs-snapshot-local clean source:jar deploy "
    }

    stage 'Deploy Web App On CloudFoundry'
    wrap([$class: 'CloudFoundryCliBuildWrapper',
        cloudFoundryCliVersion: 'Cloud Foundry CLI (built-in)',
        apiEndpoint: 'https://api.hackney.cf-app.com',
        skipSslValidation: true,
        credentialsId: 'pcf-elastic-runtime-credentials',
        organization: 'cloudbees',
        space: 'development']) {

           sh 'cf push gameoflife-dev -p gameoflife-web/target/gameoflife.war'
    }

    stash name: 'acceptance-tests', includes: 'gameoflife-acceptance-tests, gameoflife-web/target/gameoflife.war'
}

stage 'Test Web App with Selenium'
mail body: "Start web browser tests on http://gameoflife-dev.hackney.cf-app.com/ ?",subject: "Start web browser tests on http://gameoflife-dev.hackney.cf-app.com/ ?", to: 'cleclerc@cloudbees.com'
input "Start web browser tests on http://gameoflife-dev.hackney.cf-app.com/ ?"

node {
    checkpoint 'Web Browser Tests'
    unstash name: 'acceptance-tests'

    // web browser tests are fragile, test up to 3 times
    retry(3) {
        docker.image('cloudbees/java-build-tools:0.0.5').inside {
            def mavenSettingsFile = "${pwd()}/.m2/settings.xml"

            wrap([$class: 'ConfigFileBuildWrapper',
                managedFiles: [[fileId: 'maven-settings-for-gameoflife', targetLocation: "${mavenSettingsFile}"]]]) {

                sh """
                   # curl http://gameoflife-dev.hackney.cf-app.com/
                   # curl -v http://localhost:4444/wd/hub
                   # tree .
                   cd gameoflife-acceptance-tests
                   mvn -B -V -s ${mavenSettingsFile} verify -Dwebdriver.driver=remote -Dwebdriver.remote.url=http://localhost:4444/wd/hub -Dwebdriver.base.url=http://gameoflife-dev.hackney.cf-app.com/
                """
            }
        }
    }
}