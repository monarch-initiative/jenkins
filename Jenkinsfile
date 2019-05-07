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
                            sh 'sudo rm -rf ./data-graph/'
                            deleteDir()}
                        dir('./load-scigraph-data-on-dev') {
                            sh '''
                                SCIGRAPH_DIR=$WORKSPACE/load-scigraph-data-on-dev
                                
                                # Get and install the queries and curies
                                git clone https://github.com/monarch-initiative/dipper.git dipper
                                git clone https://github.com/monarch-initiative/monarch-cypher-queries.git monarch-cypher-queries
                                git clone https://github.com/SciGraph/SciGraph.git SciGraph

                                cd SciGraph && mvn clean install -DskipTests
                                cd ../dipper/maven && mvn clean install
                                cd ../../monarch-cypher-queries && mvn clean install && cd ..

                                # package docker images
                                git clone https://github.com/monarch-initiative/SciGraph-docker-monarch-data.git SciGraph-docker-monarch-data
                                cd SciGraph-docker-monarch-data && mvn clean package

                                # fire the load
                                docker run -v $SCIGRAPH_DIR/data-graph:/scigraph scigraph-load-monarch

                                # Change ownership from root to jenkins
                                sudo chown -R jenkins:jenkins $SCIGRAPH_DIR/data-graph

                                # move graph to expected neo4j dir structure
                                mkdir -p $SCIGRAPH_DIR/data-graph/databases/graph.db
                                mv $SCIGRAPH_DIR/data-graph/graph/* $SCIGRAPH_DIR/data-graph/databases/graph.db/
                                cp /opt/neo4j/conf/data-graph.conf /opt/neo4j/conf/neo4j.conf

                                # Start neo4j
                                /opt/neo4j/bin/neo4j start
                                sleep 60

                                # Delete owl:Nothing edges and node
                                cat $SCIGRAPH_DIR/monarch-cypher-queries/src/main/cypher/kg-transform/del-nothing.cql | /opt/neo4j/bin/cypher-shell -a bolt://localhost:7687

                                # Add variant labels
                                cat $SCIGRAPH_DIR/monarch-cypher-queries/src/main/cypher/kg-transform/add-variant-label.cql | /opt/neo4j/bin/cypher-shell -a bolt://localhost:7687

                                # Stop neo4j
                                /opt/neo4j/bin/neo4j stop

                                # Move data back
                                mv $SCIGRAPH_DIR/data-graph/databases/graph.db/* $SCIGRAPH_DIR/data-graph/graph/

                                # creating the archive
                                cd $SCIGRAPH_DIR/data-graph && tar czfv scigraph.tgz graph/

                                # stop the service
                                ssh monarch@$SCIGRAPH_DATA_DEV "docker stop scigraph-services"
                                ssh monarch@$SCIGRAPH_DATA_DEV "docker rm scigraph-services"

                                # delete old graph
                                ssh monarch@$SCIGRAPH_DATA_DEV "sudo rm -f /var/www/data/scigraph.tgz"
                                ssh monarch@$SCIGRAPH_DATA_DEV "sudo rm -rf /var/scigraph/graph"

                                # copy the graph over and expand it
                                scp $SCIGRAPH_DIR/data-graph/scigraph.tgz monarch@$SCIGRAPH_DATA_DEV:~
                                ssh monarch@$SCIGRAPH_DATA_DEV "sudo mv ~/scigraph.tgz /var/www/data"
                                ssh monarch@$SCIGRAPH_DATA_DEV "cd /var/scigraph && sudo tar xzfv /var/www/data/scigraph.tgz"

                                # restart the service
                                ssh monarch@$SCIGRAPH_DATA_DEV "docker run --restart=unless-stopped -v /var/scigraph:/scigraph -v /tmp/.X11-unix:/tmp/.X11-unix -e DISPLAY=unix$DISPLAY -d -p 9000:9000 --name scigraph-services scigraph-services-monarch"

                                # clean up residual docker images
                                docker rm -v $(docker ps -a -q -f status=exited) || true
                                docker rmi $(docker images -f 'dangling=true' -q) || true

                                # Remove graph
                                rm -rf $SCIGRAPH_DIR/data-graph/
                                rm -rf $SCIGRAPH_DIR/SciGraph-docker-monarch-data/
                            '''
                        }
                    },
                    "load scigraph ontology on dev": {
                        dir('./load-scigraph-ontology-on-dev') {
                            sh 'sudo rm -rf ./ontology-graph/'
                            deleteDir()}
                        dir('./load-scigraph-ontology-on-dev') {
                            sh '''
                                SCIGRAPH_DIR=$WORKSPACE/load-scigraph-ontology-on-dev
                                
                                # Get and install the queries and curies
                                git clone https://github.com/monarch-initiative/dipper.git dipper
                                git clone https://github.com/monarch-initiative/monarch-cypher-queries.git monarch-cypher-queries
                                git clone https://github.com/SciGraph/SciGraph.git SciGraph

                                cd SciGraph && mvn clean install -DskipTests
                                cd ../dipper/maven && mvn clean install
                                cd ../../monarch-cypher-queries && mvn clean install && cd ..

                                # package docker images
                                git clone https://github.com/monarch-initiative/SciGraph-docker-monarch-ontology.git SciGraph-docker-monarch-ontology
                                cd SciGraph-docker-monarch-ontology && mvn clean package

                                # fire the load
                                docker run -v $SCIGRAPH_DIR/ontology-graph:/scigraph scigraph-load-monarch-ontology

                                sudo chown -R jenkins:jenkins $SCIGRAPH_DIR/ontology-graph

                                # move graph to expected neo4j dir structure
                                mkdir -p $SCIGRAPH_DIR/ontology-graph/databases/graph.db
                                mv $SCIGRAPH_DIR/ontology-graph/graph/* $SCIGRAPH_DIR/ontology-graph/databases/graph.db/
                                cp /opt/neo4j/conf/ontology-graph.conf /opt/neo4j/conf/neo4j.conf

                                # Start neo4j
                                /opt/neo4j/bin/neo4j start
                                sleep 30

                                # Delete owl:Nothing edges and node
                                cat $SCIGRAPH_DIR/monarch-cypher-queries/src/main/cypher/kg-transform/del-nothing.cql | /opt/neo4j/bin/cypher-shell -a bolt://localhost:7687

                                # Stop neo4j
                                /opt/neo4j/bin/neo4j stop

                                # Move data back
                                mv $SCIGRAPH_DIR/ontology-graph/databases/graph.db/* $SCIGRAPH_DIR/ontology-graph/graph/

                                # creating the archive
                                sudo rm -f $SCIGRAPH_DIR/ontology-graph/scigraph.tgz
                                cd $SCIGRAPH_DIR/ontology-graph && tar czfv scigraph.tgz graph/

                                # stop the service
                                ssh monarch@$SCIGRAPH_ONTOLOGY_DEV "docker stop scigraph-services"
                                ssh monarch@$SCIGRAPH_ONTOLOGY_DEV "docker rm scigraph-services"

                                # delete old graph
                                ssh monarch@$SCIGRAPH_ONTOLOGY_DEV "sudo rm -f /var/www/data/scigraph.tgz"
                                ssh monarch@$SCIGRAPH_ONTOLOGY_DEV "sudo rm -rf /var/scigraph/graph"

                                # copy the graph over and expand it
                                scp $SCIGRAPH_DIR/ontology-graph/scigraph.tgz monarch@$SCIGRAPH_ONTOLOGY_DEV:~
                                ssh monarch@$SCIGRAPH_ONTOLOGY_DEV "sudo mv ~/scigraph.tgz /var/www/data"
                                ssh monarch@$SCIGRAPH_ONTOLOGY_DEV "cd /var/scigraph && sudo tar xzfv /var/www/data/scigraph.tgz"

                                # restart the service
                                ssh monarch@$SCIGRAPH_ONTOLOGY_DEV "docker run --restart=unless-stopped -v /var/scigraph:/scigraph -v /tmp/.X11-unix:/tmp/.X11-unix -e DISPLAY=unix$DISPLAY -d -p 9000:9000 --name scigraph-services scigraph-services-monarch-ontology"

                                # clean up residual docker images
                                docker rm -v $(docker ps -a -q -f status=exited) || true
                                docker rmi $(docker images -f 'dangling=true' -q) || true

                                rm -rf $SCIGRAPH_DIR/ontology-graph/
                                rm -rf $SCIGRAPH_DIR/SciGraph-docker-monarch-ontology/
                            '''
                        }
                    }
                )
            }
        }

        stage('Check cypher queries against new graph') {
            steps {
                dir('./check-golr-queries') {deleteDir()}
                dir('./check-golr-queries') {
                    git(
                        url: 'https://github.com/SciGraph/golr-loader.git',
                        branch: 'master'
                    )
                    sh '''
                        wget scigraph-data-dev.monarchinitiative.org/static_files/scigraph.tgz
                        tar xzfv scigraph.tgz
                        wget https://raw.githubusercontent.com/monarch-initiative/dipper/master/dipper/curie_map.yaml
                        wget https://github.com/monarch-initiative/monarch-cypher-queries/archive/master.zip

                        unzip master.zip

                        MAVEN_OPTS="-Xmx100G" mvn compile exec:java -Dexec.mainClass="org.monarch.golr.QueriesSanityCheck" -Dexec.args="graph curie_map.yaml monarch-cypher-queries-master/src/main/cypher/golr-loader/  monarch-cypher-queries-master/src/main/cypher/feature-location/"

                        rm -rf graph
                        rm scigraph.tgz
                        '''
                }
            }
        }

        stage('Create and load golr core from scigraph dev') {
            steps {
                dir('./create-and-load-golr-core') {deleteDir()}
                dir('./create-and-load-golr-core') {
                    git(
                        url: 'https://github.com/monarch-initiative/solr-docker-monarch-golr.git',
                        branch: 'master'
                    )
                    sh '''
                        SOLR_WORKSPACE=$WORKSPACE/create-and-load-golr-core
                        # clean up residual docker images
                        docker rm -v \$(docker ps -a -q -f status=exited) || true
                        docker rmi \$(docker images -f 'dangling=true' -q) || true
                        docker build --no-cache -t solr-docker-monarch-golr .

                        docker run -v $SOLR_WORKSPACE/solr:/solr solr-docker-monarch-golr

                        sudo chown -R jenkins:jenkins ./solr

                        scp solr/golr.tar monarch@$SOLR_DEV:/tmp/

                        # clean up residual docker images
                        docker rm -v \$(docker ps -a -q -f status=exited) || true
                        docker rmi \$(docker images -f 'dangling=true' -q) || true

                        # stop solr
                        ssh monarch@$SOLR_DEV "sudo service solr stop"

                        ssh monarch@$SOLR_DEV "sudo rm -rf /var/solr/data/golr"
                        ssh monarch@$SOLR_DEV "cd /var/solr/data && sudo tar xfv /tmp/golr.tar"
                        ssh monarch@$SOLR_DEV "sudo chown -R solr:solr /var/solr/data/golr"

                        # start solr
                        ssh monarch@$SOLR_DEV "sudo service solr start"
                        
                        rm -rf ./solr/
                        '''
                }
            }
        }

        stage('Load feature and search indexes') {
            steps {
                parallel(
                    "Create and load search core from scigraph dev": {
                        dir('./create-and-load-search-core') {deleteDir()}
                        dir('./create-and-load-search-core') {
                            git(
                                url: 'https://github.com/monarch-initiative/solr-docker-monarch-search.git',
                                branch: 'master'
                            )
                            sh '''
                                SOLR_WORKSPACE=$WORKSPACE/create-and-load-search-core
                                docker build --no-cache -t solr-docker-monarch-search .
                                docker run -v $SOLR_WORKSPACE/solr:/solr solr-docker-monarch-search

                                sudo chown -R jenkins:jenkins ./solr

                                scp solr/search.tar monarch@$SOLR_DEV:/tmp/

                                # clean up residual docker images
                                docker rm -v $(docker ps -a -q -f status=exited) || true
                                docker rmi $(docker images -f 'dangling=true' -q) || true

                                # stop solr
                                ssh monarch@$SOLR_DEV "sudo service solr stop"
                                    
                                ssh monarch@$SOLR_DEV "sudo rm -rf /var/solr/data/search"
                                ssh monarch@$SOLR_DEV "cd /var/solr/data && sudo tar xfv /tmp/search.tar"
                                ssh monarch@$SOLR_DEV "sudo chown -R solr:solr /var/solr/data/search"
                                
                                rm -rf ./solr/
                            '''
                        }
                    },
                    "create and load feature location core from scigraph dev": {
                        dir('./create-and-load-feature-location-core') {deleteDir()}
                        dir('./create-and-load-feature-location-core') {
                            git(
                                url: 'https://github.com/monarch-initiative/solr-docker-monarch-feature-location.git',
                                branch: 'master'
                            )
                            sh '''
                                SOLR_WORKSPACE=$WORKSPACE/create-and-load-feature-location-core
                                docker build --no-cache -t solr-docker-monarch-feature-location .
                                docker run -v $SOLR_WORKSPACE/solr:/solr solr-docker-monarch-feature-location

                                sudo chown -R jenkins:jenkins ./solr

                                scp solr/feature-location.tar monarch@$SOLR_DEV:/tmp/

                                # clean up residual docker images
                                docker rm -v $(docker ps -a -q -f status=exited) || true
                                docker rmi $(docker images -f 'dangling=true' -q) || true

                                # stop solr
                                ssh monarch@$SOLR_DEV "sudo service solr stop"

                                ssh monarch@$SOLR_DEV "sudo rm -rf /var/solr/data/feature-location"

                                ssh monarch@$SOLR_DEV "cd /var/solr/data && sudo tar xfv /tmp/feature-location.tar"

                                ssh monarch@$SOLR_DEV "sudo chown -R solr:solr /var/solr/data/feature-location"

                                # start solr
                                ssh monarch@$SOLR_DEV "sudo service solr start"
                                
                                rm -rf ./solr/
                            '''
                        }
                    }
                )
            }
        }
        stage('Diff, Archive Solr, Load Owlsim') {
            steps {
                parallel(
                    "diff and qc": {
                        dir('./diff-and-qc') {deleteDir()}
                        dir('./diff-and-qc') {
                            git(
                                url: 'https://github.com/monarch-initiative/monarch-analysis.git',
                                branch: 'master'
                            )
                            sh '''
                                export timestamp=$(date +%Y%m%d%H%M)
                                directory=data-diff-"${timestamp}"
                                mkdir $directory

                                virtualenv -p /usr/bin/python3 venv
                                venv/bin/pip install requests markdown
                                venv/bin/python3 ./monarch/monarch-data-diff.py --config ./conf/monarch-qc.json --threshold 2 --out ./data-diff-"${timestamp}"

                                grep SEVERE /var/lib/jenkins/jobs/monarch-data-pipeline/builds/$BUILD_NUMBER/log | perl -e '$pos; while(<>){chomp; if ($_ =~ m/.*clique.*/){ $pos = 1;} elsif ($pos == 1){ $_ =~ s/SEVERE: //; print "\\n$_"; $pos=2;} elsif($pos == 2){ $_ =~ s/SEVERE: //; print "\\t$_";}}' | sed '/^$/d' > ./data-diff-"${timestamp}"/clique-warnings.tsv

                                scp -r ./$directory monarch@$MONARCH_DATA_FS:/var/www/data/qc/
                            '''
                        }
                    },
                    "create solr archive": {
                        dir('./create-solr-archive') {
                            sh '''
                                ssh monarch@$SOLR_DEV "sudo rm -f /var/www/data/solr.tgz"
                                ssh monarch@$SOLR_DEV "cd /var/solr && sudo tar czfv solr.tgz data/"
                                ssh monarch@$SOLR_DEV "sudo mv /var/solr/solr.tgz /var/www/data"
                            '''
                        }
                    },
                    "create owlsim files on dev": {
                        dir('./create-owlsim-files-on-dev') {
                            git(
                                url: 'https://github.com/monarch-initiative/configs.git',
                                credentialsId: '3ca28d15-5fa8-46b1-a2ac-a5a483694f5b',
                                branch: 'master'
                            )
                        }
                        dir('./create-owlsim-files-on-dev/monarch-owlsim-data') {deleteDir()}
                        dir('./create-owlsim-files-on-dev/monarch-owlsim-data') {
                        git(
                                url: 'https://github.com/monarch-initiative/monarch-owlsim-data.git',
                                branch: 'master'
                            )
                            sh '''
                                cd .. && ./scripts/golr_exporter.py  --taxon Hs --subject_category case --object_category phenotype --dev

                                mv Hs_case_phenotype.txt ./monarch-owlsim-data/data/Cases/UDP_case_phenotype.txt
                                mv Hs_case_labels.txt ./monarch-owlsim-data/data/Cases/UDP_case_labels.txt

                                ./scripts/golr_exporter.py  --taxon Hs --subject_category disease --object_category phenotype --dev 
                                mv Hs_* ./monarch-owlsim-data/data/Homo_sapiens/

                                ./scripts/golr_exporter.py  --taxon Hs --subject_category gene --object_category phenotype --dev
                                mv Hs_* ./monarch-owlsim-data/data/Homo_sapiens/

                                ./scripts/golr_exporter.py  --taxon Mm --subject_category gene --object_category phenotype --dev
                                mv Mm_* ./monarch-owlsim-data/data/Mus_musculus/

                                ./scripts/golr_exporter.py  --taxon Dr --subject_category gene --object_category phenotype --dev
                                mv Dr_* ./monarch-owlsim-data/data/Danio_rerio/

                                ./scripts/golr_exporter.py  --taxon Dm --subject_category gene --object_category phenotype --dev
                                mv Dm_* ./monarch-owlsim-data/data/Drosophila_melanogaster/

                                ./scripts/golr_exporter.py  --taxon Ce --subject_category gene --object_category phenotype --dev
                                mv Ce_* ./monarch-owlsim-data/data/Caenorhabditis_elegans/

                                ./scripts/golr_exporter.py  --taxon Hs --subject_category publication --object_category phenotype --dev
                                grep 'PMID' Hs_publication_phenotype.txt > Pub_phenotype.txt
                                grep 'PMID' Hs_publication_labels.txt > Pub_labels.txt
                                mkdir -p ./monarch-owlsim-data/data/Pubs/
                                mv Pub_phenotype.txt Pub_labels.txt ./monarch-owlsim-data/data/Pubs/

                                cd monarch-owlsim-data

                                mkdir -p bin
                                export PATH=$PATH:$PWD/bin
                                wget http://build.berkeleybop.org/userContent/owltools/owltools -O bin/owltools
                                wget http://build.berkeleybop.org/userContent/owltools/owltools-runner-all.jar -O bin/owltools-runner-all.jar
                                chmod 755 bin/owltools

                                cd server

                                OWLTOOLS_MEMORY=50G make caches

                                # stop owlsim
                                ssh monarch@$MONARCH_APP_BETA "sudo supervisorctl stop owlsim"

                                # remove the old files
                                ssh monarch@$MONARCH_APP_BETA "rm -f /opt/owltools/ic-cache.owl"
                                ssh monarch@$MONARCH_APP_BETA "rm -f /opt/owltools/owlsim.cache"
                                ssh monarch@$MONARCH_APP_BETA "rm -f /opt/owltools/all.owl"

                                # copy new files
                                scp ic-cache.owl monarch@$MONARCH_APP_BETA:/opt/owltools
                                scp owlsim.cache monarch@$MONARCH_APP_BETA:/opt/owltools
                                scp all.owl monarch@$MONARCH_APP_BETA:/opt/owltools

                                # start owlsim
                                ssh monarch@$MONARCH_APP_BETA "sudo supervisorctl start owlsim"

                                scp -r ./monarch-owlsim-data/data monarch@$MONARCH_DATA_FS:/var/www/data/owlsim/
                                scp ic-cache.owl monarch@$MONARCH_DATA_FS:/var/www/data/owlsim/
                                scp owlsim.cache monarch@$MONARCH_DATA_FS:/var/www/data/owlsim/
                                scp all.owl monarch@$MONARCH_DATA_FS:/var/www/data/owlsim/
                            '''
                        }
                    },
                    "make monarch tsvs": {
                        dir('./make-monarch-tsvs') {deleteDir()}
                        dir('./make-monarch-tsvs') {
                            git(
                                url: 'https://github.com/monarch-initiative/monarch-analysis.git',
                                branch: 'master'
                            )
                            sh '''
                                virtualenv -p /usr/bin/python3 venv
                                venv/bin/pip install requests

                                mkdir out
                                venv/bin/python ./monarch/dump-data.py --out ./out/ --config ./conf/data-dump-conf.json

                                cd out

                                for directory in */ ; do
                                  cd $directory
                                  for file in ./* ; do
                                    grep -v 'well-known/gen' $file > tmp.tsv && mv tmp.tsv $file || rm $file tmp.tsv
                                  done
                                  md5sum * > md5sums && cd ..
                                done

                                tar -zcvf monarch_dump.tar.gz *

                                ssh monarch@$MONARCH_DATA_FS "mv /var/www/data/tsv/README /var/www/data/"

                                ssh monarch@$MONARCH_DATA_FS 'export timestamp=$(date +%s) && mkdir /tmp/tsvs-"${timestamp}" && mv /var/www/data/tsv/* /tmp/tsvs-"${timestamp}"\'

                                scp ./monarch_dump.tar.gz monarch@$MONARCH_DATA_FS:/var/www/data/tsv/

                                ssh monarch@$MONARCH_DATA_FS "tar -zxvf /var/www/data/tsv/monarch_dump.tar.gz -C /var/www/data/tsv/"
                                ssh monarch@$MONARCH_DATA_FS "rm /var/www/data/tsv/monarch_dump.tar.gz"

                                ssh monarch@$MONARCH_DATA_FS "mv /var/www/data/README /var/www/data/tsv/"
                            '''
                        }
                    }
                )
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
