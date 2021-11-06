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
                                venv/bin/python3 ./scripts/monarch-count-diff.py --config ./conf/monarch-qc.yaml -t 10 --out $directory

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
                                url: 'git@github.com:monarch-initiative/configs.git',
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
                                
                                ./scripts/golr_exporter.py  --taxon Xe --subject_category gene --object_category phenotype --dev
                                mv Xe_* ./monarch-owlsim-data/data/Xenopus/

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
                                venv/bin/python3 ./scripts/dump-data.py --solr "http://monarch-solr6-dev.cgrb.oregonstate.local/solr/golr/select" --out ./out/ --config ./conf/data-dump-conf.yaml
                                cd out

                                for directory in */ ; do
                                  cd $directory
                                  for file in ./* ; do
                                    zgrep -v 'well-known/gen' $file > tmp.tsv && gzip tmp.tsv && mv tmp.tsv.gz $file
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
