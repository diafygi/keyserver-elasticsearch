# keyserver-elasticsearch

This is the documentation for https://keyserver-elasticsearch.daylightpirates.org/

It is an elasticsearch node that contains a recent dump of the [SKS keyserver](https://sks-keyservers.net/)
pool database (I [maintain](https://sks.daylightpirates.org) a keyserver in the
pool). This pool is what is GPG uses by default to fetch public keys when using
`gpg --recv-key`. The purpose of this elasticsearch project is to let people do
data analysis on the keys in the pool.

### Document format

The keys loaded into the index are based on the output from my [openpgp.py](https://github.com/diafygi/openpgp-python)
script that converts pgp public keys to json.

https://github.com/diafygi/openpgp-python#output-formats

### Quick links

* [Total count of keys](https://keyserver-elasticsearch.daylightpirates.org/keyserver/_count?pretty=1)
* [Document format](https://keyserver-elasticsearch.daylightpirates.org/keyserver?pretty=1)
* [My public key](https://keyserver-elasticsearch.daylightpirates.org/keyserver/_search?q=roesler.cc&_source_include=key_id,packets.user_id&pretty=1)
* [All the keys with pictures](https://keyserver-elasticsearch.daylightpirates.org/keyserver/_search?q=JPEG&fields=packets.subpackets.encoding&_source_include=key_id,packets.user_id,packets.subpackets.image&pretty=1)
* [Raw json files](https://keyserver-elasticsearch.daylightpirates.org/dump/)

### Instructions for making your own

The server this runs on isn't super powerful, so if you want to run some heavy
queries, you may want to setup a local copy on your system. Here's how.

1. Setup elasticsearch on your [local machine](http://www.elastic.co/guide/en/elasticsearch/guide/master/_installing_elasticsearch.html).

2. Download openpgp.py.

```sh
mkdir ~/opengpg-python
cd ~/openpgp-python
wget https://raw.githubusercontent.com/diafygi/openpgp-python/master/openpgp.py > openpgp.py
```

3. Download the latest SKS keyserver dump (this will take a while).

```sh
mkdir ~/dump
cd ~/dump
wget -c -r -p -e robots=off --timestamping --level=1 --cut-dirs=3 \
--no-host-directories http://keyserver.mattrude.com/dump/current/
```

4. Parse keyserver dump to json gzip files (split every 1000 lines) (this will take several hours).

```sh
ls -1 ~/dump/*.pgp | \
xargs -I % sh -c "python ~/openpgp-python/openpgp.py --merge-public-keys '%' | \
split -l 1000 -d --filter 'gzip -9 > $FILE.gz' - '%.json.'"
```

5. Bulk index each gzip file into elasticsearch (this will take several hours).

```sh
ls -1 ~/dump/*.json.*.gz | \
xargs -I % sh -c "zcat '%' | \
sed '0~1 s/^/{ \"index\" : { \"_index\" : \"keyserver1\", \"_type\" : \"key\" } }\n/' | \
curl -X POST --data-binary @- http://localhost:9200/_bulk | \
{ cat -; echo ''; } >> ~/results.log"
```


