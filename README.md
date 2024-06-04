# Elastic Stack UAM Full (On-Prem)

## **Intro**

This repo contains instructions on the full set of activities to complete the User Activity Monitoring Customer Architecture Play in an On-Premise environment. The approach is broken into Kibana Auditing and Elasticsearch Auditing and combines UAM and enhanced Audit Logging. 

This guide assumes the following:
* A Platimun or Enterprise License
* Transport and HTTP security setup.
* A monitoring cluster (8.x) with trust established against the main cluster to enable CCR
* Relevant permissions to access system indices, create API and manage configuration
* Sudo access to the machines running ES and Kibana
* Filebeat instances to ship auditing data from Kibana and Elastic to the monitoring cluster


***Main Cluster***

1. Ensure security settings have been enabled in elasticsearch.yml.
*The xpack.http.ssl* settings are watcher specific and are required to run the reindexing.

Example config:

```
xpack.security.enrollment.enabled: true
xpack.security.enabled: true
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: /path-to/config/certs/elastic-certificates.p12
xpack.security.http.ssl.truststore.path: /path-to/config/certs/truststore.p12
xpack.security.http.ssl.verification_mode: certificate

xpack.http.ssl.keystore.path: /path-to/config/certs/elastic-certificates.p12
xpack.http.ssl.truststore.path: /path-to/config/certs/truststore.p12
xpack.http.ssl.verification_mode: certificate

xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.keystore.path: /path-to/config/certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: /path-to/config/certs/truststore.p12
xpack.security.transport.ssl.verification_mode: certificate

```

2. Enable auditing on every Kibana and Elastic node serving the main cluster:

* You’ll need to restart the nodes for changes to take effect. 

elasticsearch.yml:

```
xpack.security.audit.enabled: true
xpack.security.audit.logfile.events.include: "authentication_success"
xpack.security.audit.logfile.events.emit_request_body: true
xpack.security.audit.logfile.events.ignore_filters.system.users: ["*_system", "found-internal-*",  "_xpack_security", "_xpack", "elastic/fleet-server","_async_search", "found-internal-admin-proxy"]
xpack.security.audit.logfile.events.ignore_filters.realm.realms : [ "_es_api_key" ]
xpack.security.audit.logfile.events.ignore_filters.internal_system.indices : ["*ml-inference-native-*", "*monitoring-es-*"]
```

kibana.yml:

```
xpack.security.audit.enabled: true
xpack.security.audit.ignore_filters:
- categories: [web]
- actions: [saved_object_open_point_in_time, saved_object_close_point_in_time, saved_object_find, space_get, saved_object_create]
```

In the Dev Tools console of the cluster, run the following command to insert additional dynamic audit settings. (Note: These can be updated without restarting the deployment, allowing for faster iteration of tweaking and customizing audit settings. In your environment, you may have additional users you wish to ignore.)

    ```
    PUT _cluster/settings
    {
      "persistent": {
        "xpack.security.audit.logfile.events": {
          "emit_request_body": true,
          "ignore_filters.ess-internal-users.users": [
            "_system",
            "elastic/fleet-server",
            "found-internal-admin-proxy",
            "found-internal-system",
            "kibana-metricbeat"
          ],
          "include": [
            "authentication_success"
          ]
        }
      }
    }
    ```

3. In Dev Tools, set up the components and pipeline to create the kibana_objects_01 index which contains the object id to name mapping

    a) In Dev Tools Console (or via the API), create an ingest pipeline using the config from ingest-pipeline.txt.
    This ingest pipeline extracts the saved objet id from the "_id" field of documents in the kibana_analytics index and removes fields that are not required for analysis. 

    b) Create a component template using the component-template.txt file. This template contains the mapping for the new kibana objects index and speicifes use of the ingest pipeline.

    c) Create an index template that uses the component template:

    ```
    PUT _index_template/kibana_objects-new
    {
      "index_patterns": ["kibana_objects*"],
      "composed_of": ["kibana-objects"]
    }
    ```
        
    d) Reindex the existing kibana_analytics data into the new kibana index with the formlised mappings and ingest pipeline:
      ```
      POST _reindex
      {
        "source" : {
            "index" : ".kibana_analytics"
        },
        "dest" : {
            "index" : "kibana_objects-01",
            "pipeline" : "kibana-objectid"
        }
      }
      ```
4. Add an advanced watch using the watcher.txt file. This watcher checks for new documents in the kibana_analytics system index and reindexes into the new kibana index if the condition is met.

  -> You'll need to first create an API Key for authorization of the request via Stack Management-> Security API Keys or in Dev Tools Console:
  
  ```
POST /_security/api_key
{
  "name": "system_index_access_key",
  "role_descriptors": {
    "system_index_access": {
      "cluster": [],
      "index": [
        {
          "names": [".kibana_analytics*", "kibana_objects-01"],
          "privileges": ["all"],
          "allow_restricted_indices": true
        }
      ]
    }
  }
}
```

- Edit the Watcher before creating it and add in the Elasticsearch (.es.) host name and API Key. Note: When adding in the API key the line should start: "Authorization": "ApiKey xxxxxx" followed by a space and then the actual key value

5. Still in the Dev Tools console, add slowlog settings for any indices of interest. Note: consider adding these settings to index templates so future indices automatically inherit the settings upon creation.

    ```
    PUT kibana_sample_data_ecommerce/_settings
    {
        "index.search.slowlog.threshold.query.info": "0ms",
        "index.search.slowlog.threshold.fetch.info": "0ms"
    }
    ```

***Filebeat***

6. Download and install Filebeat on every machine running a main cluster Kibana/Elastic node. Filebeat should ideally be setup to run as a service. Follow: [Filebeat and systemd](https://www.elastic.co/guide/en/beats/filebeat/8.11/running-with-systemd.html).

7. Enable the Elasticsearch and Kibana modules on the relevant nodes:

```
filebeat modules enable kibana
filebeat modules enable elasticsearch
```
* For nonlinux see: Filebeat quick start: installation and configuration 

8. Modify filebeat-path-to/modules.d/elasticsearch.yml to read the Elastic auditing log files:

```
- module: elasticsearch
  audit:
    enabled: true
    var.paths:
      - /path-to-elastic/logs/elasticsearch_audit.json

  slowlog:
    enabled: true
    var.paths:
      - /path-to-elastic/logs/elasticsearch_index_search_slowlog.json
      - /path-to-elastic/logs/elasticsearch_index_indexing_slowlog.json
```

9. Modify filebeat-path-to/modules.d/kibana.yml to read the Kibana auditing log files:

```
- module: kibana
  audit:
    enabled: true
    var.paths:
      - /path-to-kibana/logs/audit.log
```

10. Modify filebeat.yml to ship data to the monitoring cluster: filebeat/filebeat.yml

Filebeat must use certs trusted by the Monitoring cluster instance. The configuration file provides an example. Alternative methods of auth / trust can be employed. See: Configure the Elasticsearch output | Filebeat Reference [8.13] | Elastic. 

Note the output will set up an index template and ILM policy conforming to the elastic-logs-8 pattern. This is so we have control over pipelines / mappings without modiying default filebeat templates.

Test the config and output to verify connection.

```
filebeat test config
filebeat test output
```

***Monitoring Deployment***
- The files needed for this section are contained in mon-cluster-side folder in this repo.

The Monitoring Cluster should be set up to trust the Main Cluster. This needs to be done at the config level. For more information, see: ​​Add remote clusters using TLS certificate authentication | Elasticsearch Guide [8.11] | Elastic

As per the Main Cluster, security settings should also include xpack.http.ssl* values to accommodate the watcher webhook action.

Configure the Main Cluster as a remote cluster:

*Modify the “seeds” value as required. Note - the port should be the transport port. You can also do this via the UI.

```
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "main-cluster": {
          "skip_unavailable": false,
          "mode": "sniff",
          "proxy_address": null,
          "proxy_socket_connections": null,
          "server_name": null,
          "seeds": [
            "127.0.0.1:9300"
          ],
          "node_connections": 3
        }
      }
    }
  }
}
```

11. Create an enrich processor using enrich.txt and execute:

    b) Execute the processor

  ```
  PUT _enrich/policy/objectid-policy/_execute
  ```

12. Create an ingest pipeline using mon-cluster-side/ingest-pipeline.txt. This utilises the new enrichment policy and set processors to set the title of the object.

13. Crete a component template to formalise mappings of the new enriched index that will be used for visualisations.

    a) Use mon-cluster-side/component-template.txt

    b) Create an index template using the new component template:

```
PUT _index_template/kibana-transform
{
  "template": {
    "settings": {
      "index": {
        "default_pipeline": "enrich-ids",
        "final_pipeline": "enrich-ids"
      }
    }
  },
  "index_patterns": ["kibana-transform-*"],
  "composed_of": ["transform-obj"]
}
```

14. Create a transform. The transform filters data from the kibana logs based on the presence of the saved_object.id field. Do not start the transform yet.

    a) Create the transform using mon-cluster-side/transform.txt

15. Create a watch that re-executes the enrich policy when new objects are added, using watcher.txt.
  -> You'll need to first create an API Key for authorization of the request via Stack Management-> Security API Keys or in Dev Tools Console:
```
POST /_security/api_key
{
  "name": "enrich_access_key",
  "role_descriptors": {
    "system_index_access": {
      "cluster": ["manage_enrich"],
      "index": [
        {
          "names": ["kibana_objects-01"],
          "privileges": ["all"],
          "allow_restricted_indices": true
        }
      ]
    }
  }
}
```

- Edit the Watcher before creating it and add in the Elasticsearch (.es.) host name and API Key. Note: When adding in the API key the line should start: "Authorization": "ApiKey xxxxxx" followed by a space and then the actual key value

16. Using the Dev Tools console, create an ingest pipeline for each pipeline in the [pipeline](audit-logging-main/pipeline) folder in this repo. The name of each pipeline must be the filename! For example:

    ```
    PUT _ingest/pipeline/stack-uam-router
    {
        "description": "Router for Stack UAM custom ETL of Elasticsearch logs. Last update 2024-01-09.",
        "version": 1,
        "processors": [
            {
                "pipeline": {
                    "name": "stack-uam-audit-extra",
                    "if": "ctx?.event?.dataset == 'elasticsearch.audit'"
                }
            },
            {
                "pipeline": {
                    "name": "stack-uam-slowlog-extra",
                    "if": "ctx?.event?.dataset == 'elasticsearch.slowlog'"
                }
            }
        ]
    } 
    ```

    Repeat the above for each pipeline. Currently, there are 3 pipelines that you must create. 

***Starting up***

17. Start filebeat as a service:
    
```
sudo service filebeat start
```

18. Verify data is being ingested:
     
    a) Create a dataview named elastic-logs-8*

    b) Start the transform

```
POST _transform/kibana-transform-01/_start
```

19. Import the following assets via Stack Management -> Saved Objects:
- assets/export.ndjson
- assets/8.11-dashboard.ndjson

5. Import the following assets via Stack Management -> Saved Objects:

    - audit-logging-main/export.ndjson
    - /Users/camillecorti-georgiou/stackuam-onprem-1/stack-uam-main/assets/8.11-dashboard.ndjson
