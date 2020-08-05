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
        BETA="https://archive.monarchinitiative.org/beta"
        WGET="wget --recursive --no-parent --no-verbose --no-host-directories --level=1 --cut-dirs=1 --reject 'index.html'"
    }

    options {
        // need plugin for this
        //timestamps()
        buildDiscarder(logRotator(numToKeepStr: '14'))
    }

    stages {

        stage('Load SciGraph instances') {
            agent { 
                label 'high_mem'
            }
            steps {
                parallel(
                    "load scigraph data on dev": {
                        dir('./load-scigraph-data-on-dev') {
                            deleteDir()
                        }
                        dir('./load-scigraph-data-on-dev') {
                            git(
                                url: 'https://github.com/monarch-initiative/scigraph-docker',
                                branch: 'master'
                            )
                            sh '''
                                git clone https://github.com/monarch-initiative/monarch-cypher-queries.git monarch-cypher-queries
                                # generate config files
                                ./conf/build-load-conf.sh data "$BETA/translationtable/curie_map.yaml"
                                ./conf/build-service-conf.sh data "$BETA/translationtable/curie_map.yaml"

                                cd ./data/
                                $WGET --accept ".nt","*_dataset.ttl" $BETA/rdf/
                                $WGET $BETA/owl/
                                chmod a+r *
                                cd ..

                                SCIGRAPH_DIR=$WORKSPACE/load-scigraph-data-on-dev

                                docker build -t scigraph-data .

                                docker run \\
                                    --volume $SCIGRAPH_DIR/data:/data \\
                                    --volume $SCIGRAPH_DIR/conf:/scigraph/conf \\
                                    scigraph-data load-scigraph monarchLoadConfiguration.yaml

                                # move graph to expected neo4j dir structure
                                mkdir -p $SCIGRAPH_DIR/data/databases/graph.db
                                mv $SCIGRAPH_DIR/data/graph/* $SCIGRAPH_DIR/data/databases/graph.db/
                                cp /opt/neo4j/conf/data-graph.conf /opt/neo4j/conf/neo4j.conf

                                # start neo4j
                                /opt/neo4j/bin/neo4j start
                                sleep 120

                                # delete owl:Nothing edges and node
                                cat $SCIGRAPH_DIR/monarch-cypher-queries/src/main/cypher/kg-transform/del-nothing.cql |
                                    /opt/neo4j/bin/cypher-shell -a bolt://localhost:7687

                                # add variant labels
                                cat $SCIGRAPH_DIR/monarch-cypher-queries/src/main/cypher/kg-transform/add-variant-label.cql |
                                    /opt/neo4j/bin/cypher-shell -a bolt://localhost:7687

                                # remove subClassOf cycles for phenotypes
                                # https://github.com/monarch-initiative/monarch-ontology/issues/21
                                cat $SCIGRAPH_DIR/monarch-cypher-queries/src/main/cypher/kg-transform/del-subclass-cycles.cql |
                                    /opt/neo4j/bin/cypher-shell -a bolt://localhost:7687

                                # stop neo4j
                                export NEO4J_SHUTDOWN_TIMEOUT=1200
                                /opt/neo4j/bin/neo4j stop

                                # move data back
                                mv $SCIGRAPH_DIR/data/databases/graph.db/* $SCIGRAPH_DIR/data/graph/

                                # creating the archive
                                cd $SCIGRAPH_DIR/data && tar czfv scigraph.tgz graph/

                                # stop the service
                                ssh monarch@$SCIGRAPH_DATA_DEV "docker stop scigraph-services"

                                # delete old graph
                                ssh monarch@$SCIGRAPH_DATA_DEV "sudo rm -f /var/www/data/scigraph.tgz"
                                ssh monarch@$SCIGRAPH_DATA_DEV "sudo rm -rf /var/scigraph/graph"

                                # copy the graph over and expand it
                                scp $SCIGRAPH_DIR/data/scigraph.tgz monarch@$SCIGRAPH_DATA_DEV:~

                                # move the graph
                                ssh monarch@$SCIGRAPH_DATA_DEV "sudo mv ~/scigraph.tgz /var/www/data"
                                ssh monarch@$SCIGRAPH_DATA_DEV "cd /var/scigraph && sudo tar xzfv /var/www/data/scigraph.tgz"

                                # move the config
                                scp $SCIGRAPH_DIR/conf/monarchConfiguration.yaml monarch@$SCIGRAPH_DATA_DEV:~
                                ssh monarch@$SCIGRAPH_DATA_DEV "sudo cp ~/monarchConfiguration.yaml /var/scigraph/conf/"
                                ssh monarch@$SCIGRAPH_DATA_DEV "sudo mv ~/monarchConfiguration.yaml /var/www/data/"


                                # start the service
                                ssh monarch@$SCIGRAPH_DATA_DEV "docker start scigraph-services"

                                # clean up residual docker images
                                docker rm -v $(docker ps -a -q -f status=exited) || true
                                docker rmi $(docker images -f 'dangling=true' -q) || true
                            '''
                        }
                    },
                    "load scigraph ontology on dev": {
                        dir('./load-scigraph-ontology-on-dev') {
                            deleteDir()
                        }
                        dir('./load-scigraph-ontology-on-dev') {
                            git(
                                url: 'https://github.com/monarch-initiative/scigraph-docker',
                                branch: 'master'
                            )
                            sh '''
                                git clone https://github.com/monarch-initiative/monarch-cypher-queries.git monarch-cypher-queries

                                # generate config files
                                ./conf/build-load-conf.sh ontology "$BETA/translationtable/curie_map.yaml"
                                ./conf/build-service-conf.sh ontology "$BETA/translationtable/curie_map.yaml"

                                cd ./data/
                                $WGET $BETA/owl/
                                chmod a+r *
                                cd ..

                                SCIGRAPH_DIR=$WORKSPACE/load-scigraph-ontology-on-dev

                                docker build -t scigraph .

                                docker run \\
                                    --volume $SCIGRAPH_DIR/data:/data \\
                                    --volume $SCIGRAPH_DIR/conf:/scigraph/conf \\
                                    scigraph load-scigraph monarchLoadConfiguration.yaml

                                # move graph to expected neo4j dir structure
                                mkdir -p $SCIGRAPH_DIR/data/databases/graph.db
                                mv $SCIGRAPH_DIR/data/graph/* $SCIGRAPH_DIR/data/databases/graph.db/
                                cp /opt/neo4j/conf/ontology-graph.conf /opt/neo4j/conf/neo4j.conf

                                # start neo4j
                                /opt/neo4j/bin/neo4j start
                                sleep 120

                                # delete owl:Nothing edges and node
                                cat $SCIGRAPH_DIR/monarch-cypher-queries/src/main/cypher/kg-transform/del-nothing.cql |
                                    /opt/neo4j/bin/cypher-shell -a bolt://localhost:7687

                                # remove subClassOf cycles for phenotypes
                                # https://github.com/monarch-initiative/monarch-ontology/issues/21
                                cat $SCIGRAPH_DIR/monarch-cypher-queries/src/main/cypher/kg-transform/del-subclass-cycles.cql |
                                    /opt/neo4j/bin/cypher-shell -a bolt://localhost:7687

                                # stop neo4j
                                export NEO4J_SHUTDOWN_TIMEOUT=600
                                /opt/neo4j/bin/neo4j stop

                                # move data back
                                mv $SCIGRAPH_DIR/data/databases/graph.db/* $SCIGRAPH_DIR/data/graph/

                                # creating the archive
                                cd $SCIGRAPH_DIR/data && tar czfv scigraph.tgz graph/

                                # stop the service
                                ssh monarch@$SCIGRAPH_ONTOLOGY_DEV "docker stop scigraph-services"

                                # delete old graph
                                ssh monarch@$SCIGRAPH_ONTOLOGY_DEV "sudo rm -f /var/www/data/scigraph.tgz"
                                ssh monarch@$SCIGRAPH_ONTOLOGY_DEV "sudo rm -rf /var/scigraph/graph"

                                # copy the graph over and expand it
                                scp $SCIGRAPH_DIR/data/scigraph.tgz monarch@$SCIGRAPH_ONTOLOGY_DEV:~
                                ssh monarch@$SCIGRAPH_ONTOLOGY_DEV "sudo mv ~/scigraph.tgz /var/www/data"
                                ssh monarch@$SCIGRAPH_ONTOLOGY_DEV "cd /var/scigraph && sudo tar xzfv /var/www/data/scigraph.tgz"

                                # move the config
                                scp $SCIGRAPH_DIR/conf/monarchConfiguration.yaml monarch@$SCIGRAPH_ONTOLOGY_DEV:~
                                ssh monarch@$SCIGRAPH_ONTOLOGY_DEV "sudo cp ~/monarchConfiguration.yaml /var/scigraph/conf/"
                                ssh monarch@$SCIGRAPH_ONTOLOGY_DEV "sudo mv ~/monarchConfiguration.yaml /var/www/data/"

                                # start the service
                                ssh monarch@$SCIGRAPH_ONTOLOGY_DEV "docker start scigraph-services"

                                # clean up residual docker images
                                docker rm -v $(docker ps -a -q -f status=exited) || true
                                docker rmi $(docker images -f 'dangling=true' -q) || true
                            '''
                        }
                    }
                )
            }
        }

        stage('Create and load golr core from scigraph dev') {
            agent { 
                label 'high_mem'
            }
            steps {
                dir('./load-golr-core') {
                    deleteDir()
                }
                dir('./load-golr-core') {
                    git(
                        url: 'https://github.com/monarch-initiative/solr-docker-monarch-golr',
                        branch: 'master'
                    )
                    sh '''
                        SOLR_DIR=$WORKSPACE/load-golr-core
                        mkdir solr
                        
                        docker build --no-cache -t solr-docker-monarch-golr .
                        docker run -v $SOLR_DIR/solr:/solr solr-docker-monarch-golr

                        # stop solr
                        ssh monarch@$SOLR_DEV "sudo service solr stop"

                        ssh monarch@$SOLR_DEV "sudo rm -rf /var/solr/data/golr"
                        ssh monarch@$SOLR_DEV "sudo chown monarch:monarch /mnt/data/jenkins/monarch-data-pipeline/load-golr-core/solr/golr.tar"
                        ssh monarch@$SOLR_DEV "cd /var/solr/data && sudo tar xfv /mnt/data/jenkins/monarch-data-pipeline/load-golr-core/solr/golr.tar"

                        ssh monarch@$SOLR_DEV "sudo chown -R solr:solr /var/solr/data/golr"

                        # start solr
                        ssh monarch@$SOLR_DEV "sudo service solr start"
                    '''
                }
            }
        }

        stage('Load feature and search indexes') {
            steps {
                parallel(
                    "Create and load search core from scigraph dev": {
                        dir('./create-and-load-search-core') {
                            deleteDir()
                        }
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

                                # start solr
                                ssh monarch@$SOLR_DEV "sudo service solr start"

                                rm -rf ./solr/
                            '''
                        }
                    },
                    "create and load feature location core from scigraph dev": {
                        dir('./create-and-load-feature-location-core') {
                            deleteDir()
                        }
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
                        dir('./diff-and-qc') {
                            deleteDir()
                        }
                        dir('./diff-and-qc') {
                            git(
                                url: 'https://github.com/monarch-initiative/release-utils.git',
                                branch: 'master'
                            )
                            sh '''
                                export timestamp=$(date +%Y%m%d%H%M)
                                directory=data-diff-"${timestamp}"
                                mkdir $directory

                                virtualenv -p /usr/bin/python3 venv
                                venv/bin/pip install -r requirements.txt
                                export PYTHONPATH=.:$PYTHONPATH
                                venv/bin/python3 ./scripts/monarch-count-diff.py --config ./conf/monarch-qc.yaml -t 10 -q

                                mv ./monarch-diff.md ./$directory/monarch-diff-"${timestamp}".md && mv ./monarch-diff.html ./$directory/monarch-diff-"${timestamp}".html

                                grep SEVERE /var/lib/jenkins/jobs/monarch-data-pipeline/builds/$BUILD_NUMBER/log |
                                    perl -e '$pos; while(<>){chomp; if ($_ =~ m/.*clique.*/){ $pos = 1;} elsif ($pos == 1){ $_ =~ s/SEVERE: //; print "\\n$_"; $pos=2;} elsif($pos == 2){ $_ =~ s/SEVERE: //; print "\\t$_";}}' |
                                    sed '/^$/d' > ./data-diff-"${timestamp}"/clique-warnings.tsv

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
                        dir('./create-owlsim-files-on-dev/monarch-owlsim-data') {
                            deleteDir()
                        }
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
                                wget http://current.geneontology.org/bin/owltools -O bin/owltools
                                chmod 755 bin/owltools

                                cd server

                                OWLTOOLS_MEMORY=100G make caches

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

                                scp ic-cache.owl monarch@$MONARCH_DATA_FS:/var/www/data/owlsim/
                                scp owlsim.cache monarch@$MONARCH_DATA_FS:/var/www/data/owlsim/
                                scp all.owl monarch@$MONARCH_DATA_FS:/var/www/data/owlsim/
                                cd .. && scp -r ./data monarch@$MONARCH_DATA_FS:/var/www/data/owlsim/
                                
                                sleep 900
                                
                                OWLSIM="https://beta.monarchinitiative.org/owlsim"
                                url="${OWLSIM}/getAttributeInformationProfile"

                                categories="${OWLSIM}/getAttributeInformationProfile?r=HP:0000924&r=HP:0000707&r=HP:0000152&r=HP:0001574&r=HP:0000478&r=HP:0001626&r=HP:0001939&r=HP:0000119&r=HP:0025031&r=HP:0002664&r=HP:0001871&r=HP:0002715&r=HP:0000818&r=HP:0003011&r=HP:0002086&r=HP:0000598&r=HP:0003549&r=HP:0001197&r=HP:0001507&r=HP:0000769"

                                until [ $(curl --write-out %{http_code} --silent --output /dev/null ${url}) -eq 200 ] ; do
                                  >&2 echo "owlsim loading"
                                  sleep 15
                                done
                                curl --silent --show-error --output /dev/null ${categories}
                            '''
                        }
                    },
                    "make monarch tsvs": {
                        dir('./make-monarch-tsvs') {
                            deleteDir()
                        }
                        dir('./make-monarch-tsvs') {
                            git(
                                url: 'https://github.com/monarch-initiative/release-utils.git',
                                branch: 'master'
                            )
                            sh '''
                                virtualenv -p /usr/bin/python3 venv
                                venv/bin/pip install -r requirements.txt
                                export PYTHONPATH=.:$PYTHONPATH

                                mkdir out
                                venv/bin/python3 ./scripts/dump-data.py --out ./out/ --config ./conf/data-dump-conf.yaml
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
