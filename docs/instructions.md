# Deploy Logsearch Boshrelease
    
You can skip right to these [instructions](https://gist.github.com/malston/02171536c1010f2bd12c) if you already know how to deploy bosh releases.

## Getting Started

First make sure you have properly targeted your existing BOSH director. Then
you can upload the latest logsearch release...

    $ mkdir -p ~/workspace/boshreleases/
    $ cd ~/workspace/boshreleases/
    $ git clone https://github.com/logsearch/logsearch-boshrelease.git
    $ cd logsearch-boshrelease/
    $ git checkout v18
    $ bosh upload release releases/logsearch-18.yml

Next you'll need to create your own deployment manifest. Right now the easiest
way to do that is by using one of the [`examples`](./examples) as a starting
point. You should only need to update the ‘director_uuid‘ property before you deploy.

    $ bosh status --uuid        # copy the output from this command
    $ vi examples/bosh-lite.yml # update the 'director_uuid' property to match the output from the previous command
    $ bosh deployment examples/bosh-lite.yml

Deploy logsearch

    $ bosh -n deploy

Logsearch should now be up and running, however Cloud Foundry doesn’t currently know anything about it. You’ll need to make a small change to your Cloud Foundry deployment manifest to instruct the CF components to forward their logs to the logsearch ingestor. Download the Cloud Foundry manifest and open it up in your favourite editor.

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

## Install Cloud Foundry Filters


## References
- [How to Integrate Elasticsearch, Logstash and Kibana (ELK) with Cloud Foundry](http://cloudcredo.com/how-to-integrate-elasticsearch-logstash-and-kibana-elk-with-cloud-foundry/)
