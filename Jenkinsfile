pipeline {

    agent any

    //triggers {
        // Run every thursday at 1pm
        //cron('H 13 * * 4')
    //}

    environment {
        // Pin dates and day to beginning of run.
        START_DATE = sh(
            script: 'date +%Y-%m-%d',
            returnStdout: true
        ).trim()

        START_DAY = sh(
            script: 'date +%A',
            returnStdout: true
        ).trim()

        MONARCH_DATA_FS = 'monarch-ttl-prod'
        SOLR_DEV = 'monarch-solr6-dev'
        SCIGRAPH_DATA_DEV = 'monarch-scigraph-data-dev'
        SCIGRAPH_ONTOLOGY_DEV = 'monarch-scigraph-ontology-dev'
        MONARCH_APP_BETA = 'monarch-app-beta'
        MONARCH4 = 'monarch4'
    }

    options {
        // need plugin for this
        //timestamps()
        buildDiscarder(logRotator(numToKeepStr: '14'))
    }

    stages {

        stage('Load SciGraph instances') {
            steps {
                parallel(
                    "load scigraph data on dev": {
                        dir('./load-scigraph-data-on-dev') {
                            git(
                                url: 'https://github.com/monarch-initiative/scigraph-docker',
                                branch: 'master'
                            )
                            sh '''
                                echo 'foobar'
                            '''
                        }
                    },
                    "load scigraph ontology on dev": {
                        dir('./load-scigraph-ontology-on-dev') {
                            git(
                                url: 'https://github.com/monarch-initiative/scigraph-docker',
                                branch: 'master'
                            )
                            sh '''
                                echo 'foobaz
                            '''
                        }
                    }
                )
            }
        }

        stage('Create and load golr core from scigraph dev') {
            steps {
                sh '''
                    REMOTE_DIR="/disk/vmpartition/data/golr-docker"

                    ssh monarch@$MONARCH4 "sudo rm -rf $REMOTE_DIR/solr-docker-monarch-golr"
                    ssh monarch@$MONARCH4 "cd $REMOTE_DIR && git clone https://github.com/monarch-initiative/solr-docker-monarch-golr.git"
                    ssh monarch@$MONARCH4 "cd $REMOTE_DIR/solr-docker-monarch-golr && docker build --no-cache -t solr-docker-monarch-golr ."
                    ssh monarch@$MONARCH4 "cd $REMOTE_DIR/solr-docker-monarch-golr && docker run -v $REMOTE_DIR/solr-docker-monarch-golr/solr:/solr solr-docker-monarch-golr"

                    # stop solr
                    ssh monarch@$SOLR_DEV "sudo service solr stop"

                    ssh monarch@$SOLR_DEV "sudo rm -rf /var/solr/data/golr"
                    ssh monarch@$SOLR_DEV "sudo chown monarch:monarch /mnt/data/golr-docker/solr-docker-monarch-golr/solr/golr.tar"
                    ssh monarch@$SOLR_DEV "cd /var/solr/data && sudo tar xfv /mnt/data/golr-docker/solr-docker-monarch-golr/solr/golr.tar"

                    ssh monarch@$SOLR_DEV "sudo chown -R solr:solr /var/solr/data/golr"

                    # start solr
                    ssh monarch@$SOLR_DEV "sudo service solr start"
                '''
            }
        }
        
    }
    /*
    post {
        success {
        }
        changed {
        }
        failure {
        }
    }*/
}
