## Ceph

![Ceph arch](http://4.bp.blogspot.com/-yupsGmuzduA/Ur2G0gUXaXI/AAAAAAAACV4/ki853OD6ATA/s1600/ceph-arch.png)

---

## ELK

![ELK Stack](img/elk_stack.png)

---

## Logging

+ *rsyslog*, *syslog-ng* to forward logs to Logstash
+ Ceph has Graylog (GELF) support  
+ store logs for later use
+ analyze logs for Alerting (Kibana, X-Pack)
+ trouble shooting/postmortem analyzing
+ predict issues with [Machine Learning](https://www.elastic.co/products/x-pack/machine-learning)

--

### Forwarding Logs

+ you can format your logs before forwarding
+ there is a tutorial for *rsyslog* how to reformat to GELF
+ Logstash has a lot of input modules
  + `syslog { }`
  + `graylog { }`

--

### Ceph GELF

+ to forward logs in GELF, update ceph.conf
  + `log_to_graylog = true`
  + `log_graylog_host = 127.0.0.1`
  + `log_graylog_port = 12201`
+ restart ceph services
+ or use [DeepSea](https://github.com/suse/deepsea)
  + has support for custom Ceph config options ([b435612](https://github.com/SUSE/DeepSea/commit/b435612))
    + `salt '*' state.apply ceph.configuration`
  + could restart services 
    + `salt-run state.orch ceph.restart`

--

### Parse and manage

+ Logstash provides methods to parse logs
+ simple alerting could be done with Logstash
+ use `grok { }` and other Filters of Logstash
  + to add fields
  + better indexing and managing your data
+ [Ceph Logstash example](https://github.com/irq0/ceph-tools/blob/master/logstash/logstash.conf)

---

## ELK cluster

+ use openSUSE ELK for traditional IT
+ use Docker on openSUSE for ELK
  + there are projects for full cluster (docker-elk, elk-docker)
  + as well as Kubernetes

---

## RGW
+ Object storage client to the ceph cluster, exposes a RESTFUL S3 & Swift API

![rgw](img/rgw.png)


---

### RGW
+ A RESTful API access to object storage, a la S3 
+ implements user accounts, acls, buckets
+ heavy ecosystem of s3/swift client tooling can be leveraged against RGW

--

### RGW
+ Supports a lot of S3 like features
  - Multipart uploads
  - Object Versioning
  - torrents
  - lifecycle
  - encryption
  - compression
  - static websites
  - metadata search...
+ From Jewel we support multisite which allows geographical redundancy

---

## ElasticSearch
> _You know, for search_

- distributed
- horizontally scalable
- schemaless
- speaks REST
- Easy Configuration without setting your hair on fire


--

##  RGW Metadata search with ES
### Motivation
+ Objects have metadata associated with them that is often interesting to analyze
+ Since it is an "object storage" you don't have any traditional filesystems tool at your disposal
+ No `du`, `df` & friends, and either way these are hard on a dist. storage system

--

### Motivation
+ Some existing support with admin API, however the problems with this:
  - returns specific metadata, not ideal for aggregation
  - no notifications when new objects/buckets/accounts are created
  - also permissions for users to access the admin API is tricky, since admin API was meant for administering
+ As an storage administrator you'd be interested in finding out for eg. the top 10 users, average object size etc., no of objects held on a user account etc.

--

### Design
+ Built atop of the multisite architecture, where data & metadata is forwarded to multiple zones
+ From Kraken, we have sync plugins
+ Allows for data & metadata to be forwarded to _external_ tiers, allows for building of:
  * Interesting solutions analyzing bucket/object/user metadata (ES for starts)
  * Backup solutions (S3/cloud sync plugin for Mimic)

--

## Elastic Sync Plugin

+ Forwards metadata from other zones onto a ES instance
+ Requires a read only zone that doesn't cater to user requests & only forwards to ES

--

## Caveats
+ ES unfortunately doesn't have an off the shelf authentication module
+ ES endpoint shouldn't be made public, accessible to the cluster administrators
+ For normal user requests, RGW itself can authenticate the user; ensures users don't see other's data
+ We have an attribute mentioning owners for an object and this is used to service user requests

--

<img src="img/rgw-es.svg" width=200%></img>

--

Eg. metadata

```json
{
        "_index" : "rgw-gold-ee5863d6",
        "_type" : "object",
        "_id" : "34137443-8592-48d9-8ca7-160255d52ade.34137.1:foo:null",
        "_score" : 1.0,
        "_source" : {
          "bucket" : "testtags213",
          "name" : "foo",
          "instance" : "null",
          "versioned_epoch" : 0,
          "owner" : {
            "id" : "user1",
            "display_name" : "user1"
          },
          "permissions" : [
            "user1"
          ],
          "meta" : {
            "size" : 7,
            "mtime" : "2017-05-04T12:54:16.462Z",
            "etag" : "7ac66c0f148de9519b8bd264312c4d64"
          }
        }
      }
```

--

### Query
```json
curl -XPOST 'localhost:9200/rgw-gold/_search?size=0&pretty' -d
{
    "aggs" : {
        "avg_size" : { "avg" : { "field" : "meta.size" } }
    }
}```

--

### Response

```json
{
  "took" : 22,
  "timed_out" : false,
  "_shards" : {
    "total" : 10,
    "successful" : 10,
    "failed" : 0
  },
  "hits" : {
    "total" : 22,
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "avg_size" : {
      "value" : 177.72727272727272
    }
  }
} ```

---

## Status in OpenSUSE
- openSUSE Factory: we already have Luminous (Ceph 12.0.2), 42.3, TW
- devel package: Filesystems:Ceph

--

## Contribute

- wiki: https://en.opensuse.org/openSUSE:Ceph
- [opensuse-ceph@opensuse.org](mailto:opensuse-ceph@opensuse.org) - Discussion of Ceph specifically on openSUSE related queries
- https://ceph.com/IRC/ - Ceph upstream community mailing lists and IRC channels
- http://lists.suse.com/mailman/listinfo/deepsea-users - DeepSea upstream mailing list. 
- https://groups.google.com/forum/#!forum/openattic-users - openATTIC upstream mailing list. 
- https://github.com/ceph/ceph - upstream Ceph sources 

--

#Backup Slides

--

### Multisite 
Disaster Recovery, CDN kind of usecase 
<section data-background-color="#ffffff">
<p align="center"><img src="img/zone-sync2.png"></p>
</section>

--

### Multisite

+ Zone -> local ceph cluster, with one or many RGWs, one of the zones needs to be master eg. aws us-east-1
  Data is replicated across zones bidirectionally, metadata always relies on master zone
+ Zonegroups -> collection of zones, zonegroups only share metadata among them

