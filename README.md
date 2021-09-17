# pds-registry-app

![🤪 Unstable integration & delivery](https://github.com/NASA-PDS/pds-registry-app/workflows/%F0%9F%A4%AA%20Unstable%20integration%20&%20delivery/badge.svg) ![😌 Stable integration & delivery](https://github.com/NASA-PDS/pds-registry-app/workflows/%F0%9F%98%8C%20Stable%20integration%20&%20delivery/badge.svg)

This application enables a Planetary Data System node to register all its data products for long term preservation and sharing with the rest of the PDS system.

It is composed of 2 services:
- registry manager (the database manager) : for details see https://github.com/NASA-PDS/pds-registry-mgr-elastic
- harvest (tools to get/prepare metadata for the registry) : for details see https://github.com/NASA-PDS/harvest
 
The purpose of this repository is to integrate together these services and package them conveniently for the users.


## 👥 Contributing

Within the NASA Planetary Data System, we value the health of our community as much as the code. Towards that end, we ask that you read and practice what's described in these documents:

-   Our [contributor's guide](https://github.com/NASA-PDS/.github/blob/main/CONTRIBUTING.md) delineates the kinds of contributions we accept.
-   Our [code of conduct](https://github.com/NASA-PDS/.github/blob/main/CODE_OF_CONDUCT.md) outlines the standards of behavior we practice and expect by everyone who participates with our software.


## Installation

See pds-engineering doc produced by mvn site, to be published.


# Updating and Releasing

The PDS uses [GitHub Actions](https://github.com/features/actions) and the [Roundup](https://github.com/NASA-PDS/roundup-action) to automate updating and releasing. The instructions that follow are kept for posterity.

Update the `pom.xml` with the version of the package (e.g. 1.0.0) and versions of services (registry, harvest) validated.

Create a tag and publish it on GitHub:

    git tag v1.0.0
    git push origin --tags
    
Prepare the package:

    mvn compile  ## get the sub-packages
    mvn linkcheck:linkcheck  ## check the link in the maven site and create a report, this requires jdk1.8 to work, later version fail.
    mvn site     ## prepare the documentation
    mvn package  ## generate the package
    mvn github-release:github-release  ## publish the package on github  # TO BE DONE REPLACE WITH CONMAN or GITHUB ACTION

    
# Docker

Information about containerizing the PDS Registry App with [Docker](https://docker.com/) follow.


## Build
```
docker image build --build-arg version_reg_app=$(git rev-parse HEAD) \
             --file Dockerfile.local \
             --tag pds_registry_app:$(git rev-parse HEAD) \
             .
```

## Run

Running consists of setting up [Elasticsearch](https://www.elastic.co/elasticsearch/), loading test data, and verifying that things work.


### Elastic Search

1. prepare `mkdir /tmp/es /tmp/output`  
    These two directories are to store the data between steps. If there is an existng elastic search database or harvest working area, then there is no need for this step.
2. fetch `docker pull elasticsearch:7.10.1`  
    Only need to fetch the image once.
3. prepare a network with `docker network create pds` if not done previously
4. run  
    ```
    docker container run --detach \
               --env "discovery.type=single-node" \
               --name es \
               --network pds \
               --publish 9200:9200 \
               --publish 9300:9300 \
               --rm \
               --user $UID \
               --volume /tmp/es:/usr/share/elasticsearch/data \
               elasticsearch:7.10.1
    ```  
    Start the elastic search engine and make sure that any of the data created is owned by yourself so that you can clean it up. If you are not using the /tmp/es directory created above, then make sure to change the --volume argument as needed so that elastic search has access to it. Also change the --user arguement such that elastic search has full privledges to read, write, and modify the data.


### Install Test Data

This section will use the small test data set contained within the docker image. The steps used to do this can be used as a template to install your own more complicated datasets as well.

First, install the registry schema for elastic search. Things to note, the URL host name is the --name argument value used when starting elastic search.

```
docker container run --network pds \
           --rm \
           --user $UID \
           pds_registry_app:$(git rev-parse HEAD) \
              registry-manager create-registry -es http://es:9200 \
                                               -schema /var/local/registry/elastic/registry.json 
```

Second we need to harvest the test data. It is currently setup in /var/local/harvest/archive. If one wanted to use their own archive of data, they would simply have to add `--volume /path/to/archive:/var/local/harvest/archive` to the docker run portion of this command. Because we cannot easily run two commands in one docker run invokation, it is necessary to use an external directory for /var/local/havest/output. The ephemeral nature of a container erases what we would write into /var/local/harvest/output between the two docker run invokations.
```
docker container run --network pds \
           --rm \
           --user $UID \
           --volume /tmp/output:/var/local/harvest/output \
           pds_registry_app:$(git rev-parse HEAD) harvest \
              -c /var/local/harvest/conf/examples/bundles.xml \
              -o /var/local/harvest/output
```

Third, and last, we need to now ingest the havested data from above into elastic search. Things to note: One, the URL host name is the --name argument value used when starting elastic search. Two, the output volume must be the same mapping as in the previous step.
```
docker container run --network pds \
           --rm \
           --user $UID \
           --volume /tmp/output:/var/local/harvest/output \
           pds_registry_app:$(git rev-parse HEAD) \
           registry-manager load-data \
              -es http://es:9200 \
              -dir /var/local/harvest/output
```


### Verification

The verification script is written in Python 3.x with requests installed as well only because that is what the person doing the work knows the best. A simple curl then lots of scrolling and eyeballing can do the same thing.

```console
$ python3 <<EOF
import requests
response = requests.get('http://localhost:9200/registry/_search?q=*')
if response.status_code == 200:
    print ('successfully found elastic search')
    data = response.json()

    if not data['timed_out']:
        print ('successfully searched for query')

        if data['hits']['total']['value'] == 17:
            print ('successfully found all expected data')
        else: print ('FAILED to find all expected data')
    else: print ('FAILED query search')
else: print ('FAILED to find elastic search')
EOF
```
Output should look like this if everything is working:
```
successfully found elastic search
successfully searched for query
successfully found all expected data
```
