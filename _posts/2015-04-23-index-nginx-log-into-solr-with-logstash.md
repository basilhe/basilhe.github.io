---
layout: post
title: "Index nginx log into Solr with LogStash"
description: "Index nginx log into Solr with LogStash"
category: solr
tags: [solr,logstash]
---
{% include JB/setup %}


# LogStash

* [LogStash](http://www.logstash.net/)
* [Solr HTTP Support](http://www.logstash.net/docs/1.4.2/outputs/solr_http)
* [Contrib Plugins](http://www.logstash.net/docs/1.4.2/contrib-plugins)
* [Getting Started with LogStash](http://www.logstash.net/docs/1.4.2/tutorials/getting-started-with-logstash)

# Code 

## pattern

### sample data, /opt/apilog/api.access.log

	166.170.0.90 - - [21/Apr/2015:23:59:12 +0800] "POST /v1/provision/ HTTP/1.1" "386" 200 56 "-" "android-async-http/1.4.1 (http://loopj.com/android-async-http)" "-" "app_key=2b18563ad04411e38c5500163e0e2e0d&unique_id=10%3AA5%3AD0%3AD6%3A29%3ADE"
	166.170.5.26 - - [21/Apr/2015:23:59:12 +0800] "POST /v1/provision/ HTTP/1.1" "386" 200 56 "-" "android-async-http/1.4.1 (http://loopj.com/android-async-http)" "-" "app_key=2b18563ad04411e38c5500163e0e2e0d&unique_id=10%3AA5%3AD0%3AFE%3ADA%3AA4"
	71.105.236.33 - - [21/Apr/2015:23:59:12 +0800] "POST /v1/provision/ HTTP/1.1" "325" 200 56 "-" "android-async-http/1.4.1 (http://loopj.com/android-async-http)" "-" "app_key=2b18563ad04411e38c5500163e0e2e0d&unique_id=CC%3A3A%3A61%3A42%3AA7%3A4A"
	211.143.134.194 - - [21/Apr/2015:23:59:12 +0800] "GET /dev/jd/product/4b373b65faab4633af7de4411fc516ce HTTP/1.1" "148" 200 30 "-" "-" "-" "-"

### $logstash_home/patterns/nginx

	NGINXACCESS %{IPORHOST:clientip} %{USER:ident} %{USER:auth} \[%{HTTPDATE:timestamp}\] \"(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})\"(?: "%{NUMBER:requestlength}")? %{NUMBER:response} (?:%{NUMBER:bytes}|-) (?:%{QS:referrer}|-) (?:%{QS:agent}|-) (?:%{QS:xforwardfor}|-) (?:%{QS:body}|-)
	
	TIMESTAMP_ERROR %{YEAR}/%{MONTHNUM}/%{MONTHDAY} %{HOUR}:?%{MINUTE}(?::?%{SECOND})?
	NGINXERROR %{TIMESTAMP_ERROR:timestamp} \[error\] (?:%{DATA:reason})?, client: %{IPORHOST:clientip}, server: %{IPORHOST:server}, request: \"(?:%{WORD:verb} %{URIPATH:request}(?:%{URIPARAM:query})?(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})\"
	
### configuration, nginx.conf

	input {
      file {
        path => "/opt/apilog/api.access.log"
        start_position => beginning
      }
    }

    filter {
      mutate {
       replace => { "type" => "nginx_access" }
       gsub => [
         "message", "\"-\"", "-",
         "path", "(\/opt\/apilog\/gw)|(\.log.*$)", ""
       ]
      }
      grok {
        match => { "message" => "%{NGINXACCESS}" }
      }
      date {
        match => [ "timestamp" , "dd/MMM/YYYY:HH:mm:ss Z", "YYYY/MM/dd HH:mm:ss" ]
      }
      grok {
    	match => { "request" => "/%{GVER:ver}/%{GAPP:app}/%{GID:did}/firmwares/0" }
    	match => { "request" => "/%{GVER:ver}/%{GAPP:app}/%{GID:uid}/reset" }
    	match => { "request" => "/%{GVER:ver}/%{GAPP:app}/%{GID:product_key}/firmwares/%{BASE16NUM:fid}" }
    	match => { "request" => "/%{GVER:ver}/%{GAPP:app}/%{GSUBAPP:subapp}" }
    	match => { "request" => "/%{GVER:ver}/%{GAPP:app}" }
    	match => { "request" => "/dev/%{GAPP:app}/%{GID:did}" }
    	match => { "request" => "/dev/%{GAPP:app}/%{GAPP:subapp}/(?:%{GID:product_key}|.*)" }
    	match => { "request" => "/%{GAPP:app}/%{GAPP:subapp}/%{GID:did}(?:/%{GAPP:action})?" }
    	match => { "request" => "/%{GAPP:app}/%{GSUBAPP:subapp}(/)?" }
  	  }
      geoip {
        source => "clientip"
      }
      uuid {
        target => "@id"
      }
      urldecode {
        field => "body"
      }
      kv {
        source => "body"
        field_split => "&"
        trim => "\""
        trimkey => "\""
        include_keys => [ "app_key", "unique_id" ]
      }
      kv {
    	source => "query"
    	field_split => "&"
	    trim => "\""
    	trimkey => "?"
	    include_keys => [ "product_key", "did", "uid" ]
      }
      mutate {
        add_field => [ "country_name" , "%{[geoip][country_name]}" ]
        add_field => [ "region_name" , "%{[geoip][real_region_name]}" ]
        add_field => [ "city_name" , "%{[geoip][city_name]}" ]
        add_field => [ "continent_code" , "%{[geoip][continent_code]}" ]
        add_field => [ "country_code2" , "%{[geoip][country_code2]}" ]
        add_field => [ "country_code3" , "%{[geoip][country_code3]}" ]]
        add_field => [ "latitude" , "%{[geoip][latitude]}" ]
        add_field => [ "longitude" , "%{[geoip][longitude]}" ]
        remove_field => ['[geoip][city_name]','[geoip][continent_code]','[geoip][country_code3]','[geo
    ip][country_code2]','[geoip][country_name]','[geoip][dma_code]','[geoip][latitude]','[geoip][longi
    tude]','[geoip][postal_code]','[geoip][real_region_name]', '[geoip][area_code]', '[geoip][ip]', '[
    geoip][locaiton]', '[geoip]', '[ident]', '[auth]', '[host]', '[path]', '[requestlength]', '[body]'
    , 'timestamp', 'message', '@version', 'rawrequest', 'fid', 'tags' ]
      }
      mutate {
        convert => [ "response", "integer" ]
        convert => [ "bytes", "integer" ]
        convert => [ "latitude", "float" ]
        convert => [ "longitude", "float" ]
        gsub => [
          "referrer", "\"", "",
          "xforwardfor", "\"", "",
          "agent", "\"", "",
          "country_name", "%\{\[geoip\].*", "",
      	  "region_name", "%\{\[geoip\].*", "",
          "city_name", "%\{\[geoip\].*", "",
          "continent_code", "%\{\[geoip\].*", "",
          "country_code2", "%\{\[geoip\].*", "",
          "request", "/[a-zA-Z0-9]{32}", "/product_key",
          "request", "/[a-zA-Z0-9]{22}", "/did",
          "country_code3", "%\{\[geoip\].*", ""
        ]
      }
    }

    output {
      #elasticsearch {
        #host => "localhost"
        #protocol => "http"
      #}
      solr_http {
    	solr_url => "http://host:8983/solr/apilog_shard1_replica1"
    	workers => 2
  	  }
      #stdout { codec => rubydebug }
    }
    
## Solr Schema

	<field name="id" type="string" indexed="true" stored="true" required="true" multiValued="false"
    />
    <field name="timestamp" type="tdate" indexed="true" stored="true" />
    <field name="type" type="text_en" indexed="true" stored="true" />
    <field name="clientip" type="string" indexed="true" stored="true" />
    <field name="verb" type="string" indexed="true" stored="true" />
    <field name="request" type="string" indexed="true" stored="true" />
    <field name="httpversion" type="string" indexed="true" stored="true" />
    <field name="response" type="tint" indexed="true" stored="true" />
    <field name="bytes" type="tint" indexed="true" stored="true" />
    <field name="referer" type="string" indexed="true" stored="true" />
    <field name="agent" type="string" indexed="true" stored="true" />
    <field name="xforwardfor" type="string" indexed="true" stored="true" />

    <field name="country_name" type="string" indexed="true" stored="true" />
    <field name="region_name" type="string" indexed="true" stored="true" />
    <field name="city_name" type="string" indexed="true" stored="true" />
    <field name="continent_code" type="string" indexed="true" stored="true" />
    <field name="country_code2" type="string" indexed="true" stored="true" />
    <field name="country_code3" type="string" indexed="true" stored="true" />
    <field name="latitude" type="float" indexed="true" stored="true" />
    <field name="longitude" type="float" indexed="true" stored="true" />

    <field name="_version_" type="long" indexed="true" stored="true"/>
    
# Solr

## Runtime Solr Configuration

Configuration files for a collection are managed as part of the instance directory. To generate a skeleton of the instance directory, run the following command:
	$ solrctl instancedir --generate $HOME/solr_configsYou can customize it by directly editing the solrconfig.xml and schema.xml files created in $HOME/solr_configs/conf.

These configuration files are compatible with the standard Solr tutorial example documents.After configuration is complete, you can make it available to Solr by issuing the following command, which uploads the content of the entire instance directory to ZooKeeper:  	$ solrctl instancedir --create apilog $HOME/solr_configsUse the solrctl tool to verify that your instance directory uploaded successfully and is available to ZooKeeper. List the contents of an instance directory as follows:	$ solrctl instancedir --list

## Creating Your First Solr Collection
By default, the Solr server comes up with no collections. Make sure that you create your first collection using the instancedir that you provided to Solr in previous steps by using the same collection name. numOfShards is the number of SolrCloud shards you want to partition the collection across. The number of shards cannot exceed the total number of Solr servers in your SolrCloud cluster:	$ solrctl collection --create collection1 -s {{numOfShards}}
## Adding Another Collection with Replication
To support scaling for the query load, create a second collection with replication. Having multiple servers with replicated collections distributes the request load for each shard. Create one shard cluster with a replication factor of two. Your cluster must have at least two running servers to support this configuration, so ensure Cloudera Search is installed on at least two servers. A replication factor of two causes two copies of the index files to be stored in two different locations.1. Generate the config files for the collection:		$ solrctl instancedir --generate $HOME/solr_configs22. Upload the instance directory to ZooKeeper:		$ solrctl instancedir --create collection2 $HOME/solr_configs23. Create the second collection:		$ solrctl collection --create collection2 -s 1 -r 24. Verify that the collection is live and that the one shard is served by two hosts. For example, for the server myhost.example.com, you should receive content from: http://myhost.example.com:8983/solr/#/~cloud.## Creating Replicas of Existing ShardsYou can create additional replicas of existing shards using a command of the following form:For example to create a new replica of collection named collection1 that is comprised of shard1, use the following command:Adding a New Shard to a Solr ServerYou can use solrctl to add a new shard to a specified solr server.	solrctl --zk <zkensemble> --solr <target solr server> core --create <new core name> -p collection=<collection> -p shard=<shard to replicate>	￼￼￼￼￼solrctl --zk myZKEnsemble:2181/solr --solr mySolrServer:8983/solr core --create collection1_shard1_replica2 -p collection=collection1 -p shard=shard1	solrctl --solr http://<target_solr_server>:8983/solr core --create <core_name> -p dataDir=hdfs://<nameservice>/<index_hdfs_path> -p collection.configName=<config_name> -p collection=<collection_name> -p numShards=<int> -p shard=<shard_id>
	
Update cmds	
	vi /root/apilog_solr_configs/conf/schema.xml	solrctl instancedir --update apilog /root/apilog_solr_configs/
	solrctl collection --reload apilog
	solrctl collection --deletedocs apilog	

# References

* [Setting up Logstash 1.4.2 to forward Nginx logs to Elasticsearch](http://www.bravo-kernel.com/2014/12/setting-up-logstash-1-4-2-to-forward-nginx-logs-to-elasticsearch/)
* [Grok Constructor](http://grokconstructor.appspot.com/do/match)
* [Collect & visualize your logs with Logstash, Elasticsearch & Redis](http://michael.bouvy.net/blog/en/2013/11/19/collect-visualize-your-logs-logstash-elasticsearch-redis-kibana/)
