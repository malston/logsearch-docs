# Deploy Logsearch Boshrelease
    
You can skip right to these [instructions](https://gist.github.com/malston/02171536c1010f2bd12c) if you already know how to deploy bosh a release.

## Getting Started

If you deploying to an IaaS such as [vSphere](https://github.com/pivotalservices/logsearch-boshrelease/blob/experiment-mt/examples/micro-logsearch-vsphere.yml), then you'll need to target your microbosh director.

    $ bosh target MICRO_BOSH_IP

Otherwise, deploy [bosh-lite](https://github.com/cloudfoundry/bosh-lite) and target the director. 

    $ mkdir -p ~/workspace
    $ cd ~/workspace
    $ git clone https://github.com/cloudfoundry/bosh-lite.git
    $ cd ~/workspace/bosh-lite
    $ vagrant box update
    $ vagrant up --provider=virtualbox
    $ export no_proxy=192.168.50.4,xip.io
    $ bosh target 192.168.50.4 lite
    $ bosh login
    $ ./bin/add-route
    $ ./bin/provision_cf    # install the latest cf-release

Make sure you have properly targeted your existing BOSH director. Then you can create and upload a dev release. 

    $ mkdir -p ~/workspace/boshreleases/
    $ cd ~/workspace/boshreleases/
    $ git clone https://github.com/logsearch/logsearch-boshrelease.git
    $ cd logsearch-boshrelease/
    $ bosh create release --force
    $ bosh upload release
    
If you want to upload the latest production logsearch release, instead of running the last 2 lines above just run

    $ bosh upload release releases/logsearch-latest.yml

If you want to upload a specific release

    $ git checkout v18
    $ bosh upload release releases/logsearch-18.yml

Next you'll need to create your own deployment manifest. Right now the easiest
way to do that is by using one of the [`examples`](https://github.com/pivotalservices/logsearch-boshrelease/blob/experiment-mt/examples) as a starting
point. You should only need to update the `‘director_uuid‘` property before you deploy.

    $ bosh status --uuid        # copy the output from this command
    $ vi examples/bosh-lite.yml # update the 'director_uuid' property to match the output from the previous command
    $ bosh deployment examples/bosh-lite.yml

Deploy logsearch

    $ bosh -n deploy

Logsearch should now be up and running, however Cloud Foundry doesn’t currently know anything about it. You’ll need to make a small change to your Cloud Foundry deployment manifest to instruct the CF components to forward their logs to the logsearch ingestor. Download the Cloud Foundry manifest and open it up in your favorite editor.

    $ bosh download manifest cf-warden /tmp/cf-warden.yml
    $ vi /tmp/cf-warden.yml
    
In the `‘properties‘` section of the manifest you’ll find a property named `‘syslog_daemon_config‘`. This property needs to be updated with information about the logsearch ingestor, as follows:
```
syslog_daemon_config:
  address: 10.244.2.14
  port: 5515
  transport: relp
```
Note that the address value is the static IP address of the logsearch ingestor VM. I’ve chosen ‘relp‘ as the transport method here, but you could also use ‘syslog‘ or ‘lumberjack‘ if you’d prefer. Also note that if you’re running an earlier version of Cloud Foundry, this property may be called `‘syslog_aggregator‘` rather than `‘syslog_daemon_config‘`.

Once you’ve made the change, save the manifest file and then re-deploy Cloud Foundry.

    $ bosh deployment /tmp/cf-warden.yml
    $ bosh -n deploy

### Adding to an existing Cloud Foundry + Logsearch deployments

Logsearch for Cloud Foundry is an add-on that customizes Logsearch to work with Cloud Foundry data. 

To get this add-on you can either checkout the github repo and follow the [README](https://github.com/logsearch/logsearch-for-cloudfoundry/blob/master/logsearch-for-cloudfoundry-boshrelease/README.md) instructions to create a release.

Or, download the one from s3 which has been tested on cf-release v205 and logsearch-boshrelease v19 as follows.

0.  Deploy the `ingestor_cloudfoundry` job to your existing logsearch deployment.

  * `bosh upload release https://logsearch-for-cloudfoundry-boshrelease.s3.amazonaws.com/boshrelease-logsearch-for-cloudfoundry-0%2Bdev.3.tgz`
  * Add and configure the `ingestor_cloudfoundry` job to your logsearch deploy manifest:
           releases:
  	          - name: logsearch-for-cloudfoundry
                version: latest    
  
           jobs:
             - name: ingestor_cloudfoundry
               release: logsearch-for-cloudfoundry
               templates: 
               - name: ingestor_cloudfoundry-firehose
               instances: 1
               resource_pool: small_z1
               networks: z1
               persistent_disk: 0
  
           properties:
               ingestor_cloudfoundry-firehose:
                 debug: true
                 uua-endpoint: "https://uaa.10.244.0.34.xip.io/oauth/authorize"
                 doppler-endpoint: "wss://doppler.10.244.0.34.xip.io"
                 skip-ssl-validation: true
                 firehose-user: admin
                 firehose-password: admin
                 syslog-server: "10.244.10.6:514"
   
   * Include `logsearch-for-cloudfoundry/logstash-filters-default.conf` log_parsing rules
           properties:
             logstash_parser:
           <% filtersconf = File.join(File.dirname(File.expand_path(__FILE__)), 'path/to/logsearch-for-  cloudfoundry/logstash-filters-default.conf') %>
                filters: |
                        <%= File.read(filtersconf).gsub(/^/, '            ').strip %>

   * `bosh deploy`
   * All app logs from your CF deployment should now be forwarded into your logsearch cluster.  Find them by searching for `@type:cloudfoundry_doppler`.  Make useful dashboards like the below:
   ![screen shot 2015-03-30 at 12 48 38](https://cloud.githubusercontent.com/assets/227505/6895741/236ac118-d6db-11e4-802d-19f548d323f5.png)


## References
- [Official Logsearch Docs](http://www.logsearch.io/docs/)
- [How to Integrate Elasticsearch, Logstash and Kibana (ELK) with Cloud Foundry](http://cloudcredo.com/how-to-integrate-elasticsearch-logstash-and-kibana-elk-with-cloud-foundry/)
- [Logsearch for Cloud Foundry](https://github.com/logsearch/logsearch-for-cloudfoundry)
