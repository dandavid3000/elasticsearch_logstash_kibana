>  This guide is built in order to help people have less headache in configuration for logstash, elastic search, and kibana.
> couchdb -> logstash -> elasticsearch -> kibana

## Table of contents
* [Introduction](#introduction)
* [Installation](#installation)
* [Logstash Configuration](#logstash-configuration)
* [FAQ](#faq)
* [Conclusion](#conclusion)


## Introduction

I'm not going to focus on how to install `Logstash`, `Elasticsearch`, `Kibana` as well as `couchdb`.
`Logstash` is used to get data from `couchdb` continuously. The data then is sent to `ElasticSearch` and viewed on `Kibana`.

The fatal thing is how to get data correctly from `couchdb` using `Logstash`.

All services include: 
	* An instance running couchdb (AWS)
	* A new instance to install `ELK` (bitnami is used)
	* `Tera Term`/`putty` and `WinSCP`

You must be familiar with Linux commands to follow the guide



## Installation
* Login to your bitnami account at `https://app.bitnamihosting.com`

* Create a new server:
	* `Launch Regular Server`
	* Put `Name` and `Domain Name` if they're neccessary
	* Choose `ELK` image in `Select Application`
	* Decide `OS version` and server configuration as you wish
	* Click `Build and Launch` and wait for a couple of minutes for the server to be created

* Server access
	* Click on your server and choose `Manage Server`
	* Put `IP Address` on your terminal tool to login in
	* WinSCP is convenient for file modification/copy/paste.
	
	![Success login](/img/1.PNG "Success Login")
	
## Logstash Configuration

* Check all services running on the server by using the command:
	* get sudo permission `sudo su`
	* `/opt/bitnami/ctlscript.sh status` -> Get all services status. It can be replaced by stop/start
	
* It's neccessary to stop Logstash service before configuring anything.
	* Do `/opt/bitnami/ctlscript.sh stop logstash`
	* Recheck again with status command `/opt/bitnami/ctlscript.sh status logstash`
	
* All logs files will be stored in `/opt/bitnami/logstash/logs/`
	* `logstash.log` stores operation for logstash service (Reset everytime a logstash service starts)
	* `logstash-plain.log` stores everything relating to logstash.
	* You have to make sure that when you run logstash. There is no error while running the service.
	
* Use `chmod 777` to set permission for all files/directories in logstash folder

* Create a new folder to store couchdb seq
```
	mkdir /home/logstash/
	chmod 777 /home/logstash/
```

* Install `couchdb_changes`
```
/opt/bitnami/logstash/bin/logstash-plugin install logstash-input-couchdb_changes

```

* Modify the configuration for logstash service by opening file `/opt/bitnami/logstash/config/logstash.yml'
```
# path.config: "/opt/bitnami/logstash/conf/"

# config.test_and_exit: true

# config.reload.automatic: true

```

* Open file  `/opt/bitnami/logstash/conf/logstash.conf`. As default. Logstash gets all records in couchdb so we need to serperate the doc is `deleted` or `index`. You can read more congifuration for logstash

```
input { 
	couchdb_changes {
		db => "your_db_name"
		host => "your_db_ip_address"
		port => 5984
		username => "db_username"
		password => "db_password"
		keep_id => true
   		keep_revision => true
		initial_sequence => 0
	}
}

filter {
  if [@metadata][action] == "delete" {
      mutate {
        add_field => { "elastic_action" => "delete" }
      }
    } else {
      mutate {
        add_field => { "elastic_action" => "index" }
      }
    }
}

output {
 	elasticsearch {
		hosts => "localhost:9200"
		index => "your_index"
  		document_id => "%{[@metadata][_id]}"
   		action => "%{elastic_action}"
	}
}

```

* This configuration helps to get all documents without modifying anything from your couchdb

* Create your index with elasticsearch:
	* `curl -XPUT 'localhost:9200/your_index?pretty'` : Create.
	* `curl 'localhost:9200/_cat/indices?v'` : Show indexes
	* `curl -XDELETE localhost:9200/your_index`: Delete your_index

* Modify limit number for indexes in Elasticsearch from `1000` to `100000`
```
curl -XPUT 'localhost:9200/your_index/_settings' -H 'Content-Type: application/json' -d'
{
	"index" : {
		"mapping" : {
			"total_fields" : {
				"limit" : "100000"
			}
		}
	}
}'
```

* Double check the ElasticSearch setting again: `curl -XGET 'localhost:9200/your_index/_settings?pretty'`
	
* Start logstash service by using `/opt/bitnami/ctlscript.sh start logstash`. Remember to check log files and status again to ensure everything is working OK.

* If you're doing it correctly. You will see `docs.count` increasing time by time by using the command `curl 'localhost:9200/_cat/indices?v'`

* You'd see your index in `Management` in `Kibana`. Create indexes from it to finish and then you're able to search what you need

## FAQ

* How to get rid of `Can't run logstash service because another instance is running ...`?
	* 1st solution: Try to stop logstash service and check status.
	* 2nd solution: Delete the configure file. Create a new one and run it.
	* 3rd solution: Delete the configuration file. Stop/Start instance (server) again. This should solve the issue
	
* Is my data from couchdb transferred continously to logstash, and how long?
	* Yes: You can configure those parameters in `logstash.yml`

## Conclusion

This guide is a simple one to help you understand the big picture and quickly connect those services. You may need to take a look at logstash structure and configuration on their website for deep configuration and versatile setting.

![Success index import](/img/2.png "Success index import")


