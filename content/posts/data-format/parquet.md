+++
author = "Wen Tat"
title = "Parquet"
date = "2024-04-28"
description = "My notes about Apache Parquet"
summary = ""
tags = ["Parquet"]
categories = ["data-format"]
series = ["data-format"]
ShowToc = true
TocOpen = true
draft = true
+++

# Introduction
Apache Parquet is a free and open-source column-oriented data storage format in the Apache Hadoop ecosystem. It is a top level project of Apache and widely used in big data processing. There is a sister project of Parquet called Apache Arrow, which is in memory, but they can be converted vice-versa.

In reality, parquet refers to a type of flooring pattern, which lays the pieces of wood in horizontal and vertical patterns, the reasoning behind the name will soon become clear. 

## Motivations behind parquet:
There is the human readable JSON, XML/HTML, YAML or CSV/TSV format, but these formats are not efficient as they are stored as text which takes up a lot of bytes (even boolean values like true will be stored as 4 characters). The better formats will be the ones in binary such as protobuf or specialized binary files as the data can be stored in their native datatypes and definitions can be separated from the data. 
For binary data formats like MySQL files or protobuf files, we can do better by storing the data in a columnar format such as ORC and Parquet, so that analytics SQL can read the related data faster as well as enable better compression. 
for ORC vs Parquet, their design is roughly similar, but Parquet is better supported and optimized for SPARK engine while ORC is created for Hive

## Parquet in theory
### What is inside a parquet file
1. File
A single file of the dataset, such as compact-06999-8c7e5bc0-fa44-4ba6-b273-1c13f811665a.c000.zstd.parquet
A file has many rowgroups, and also has a footer which records metadata information
The file is usually compressed with an algorithm such as zstd, snappy, gzip etc.
Rowgroup
A chunk of bytes which has many column chunks (one for each columns) for a subset of rows
Column chunk
A chunk of bytes which contains the data for a particular column for a subset of rows
It may split the column into further smaller chunks called pages depending on the size of the column, e.g bigger columns may have more pages
Page
A page contains a group of rows
Page row
Each row in page contains the definition level, repetition level of this row as well as its value
The value can be encoded in different type of encodings, such as dictionary, plain encoding, hybrid encoding etc.
Dremel encoding method - the 1st layer
Reference: Dremel made simple with Parquet (twitter.com)
Parquet internally uses the dremel encoding method, which describes a way to store nested and repeated data in a flattened structure using repetition levels and definition levels. You can think of repetition levels and definition levels as GPS coordinates that can describe each element's position perfectly. Lets take a fake rule engine data as an example:
record 1
[
    {
        "strategy_id": 1000066,
        "rules": [
            {
                "rule_id": 1,
            },
            {
                "rule_id": 2,
            }
        ]
    },
    {
        "strategy_id": 1000067
    },
    {
        "strategy_id": 1000069,
        "rules": []
    },
    {
        "strategy_id": 1000070,
        "rules": [{}]
    }, 
]

record 2 
[]

record 3
null


In Parquet, the schema will be encoded as below
message strategies { <-- definition level 0, repetition level 0
  repeated group list { <-- repetition level 1
    optional group element { <-- definition level 1
      optional int32 strategy_id;  <-- definition level 2
      optional group rules (LIST) {  <-- definition level 3
        repeated group list {   <-- repetition level 2
          optional group element { <-- definition level 4
            optional int rule_id; <-- definition level 5
          }
        }
      }
    }
  }
}

So for column strategies.element.rules.element.rule_id, it has max_definition_level of 5 and max_repetition_level of 2

struct
value
repetition level
definition level
explanation
Record 1
 {
        "strategy_id": 1000066,
        "rules": [
            {
                "rule_id": 1,
            },
            {
                "rule_id": 2,
            }
        ]
    },
1
0
5
rule_id is defined, and this is the first value of the record, so we make it repetition level 0
    {
        "strategy_id": 1000067,
        "rules": [
            {
                "rule_id": 1,
            },
            {
                "rule_id": 2,
            }
        ]
    },
2
2
5
rule_id is defined, and this is inserted into 2nd repetition level
 {
        "strategy_id": 1000069,
        "rules": null
    },
null
1
1
rule id is null, and the max defined level is the element of the strategies list, which is definition level 1
{
        "strategy_id": 1000070,
        "rules": []
    },
null
1
3



        "strategy_id": 1000072,
        "rules": [{"rule_id": null}]
    },
null
1
4




Note:
Under the Dremel encoding, we only store the values which has definition level = max definition level, the grey out values are not recorded but shown for clarity
Dremel encoding is a shred and assembly algorithm, we can not only break down but reassemble the records with Dremel.
Column encoding methods - the 2nd layer
Reference: Encodings | Apache Parquet
As you can see with the layout of a single parquet file - the repetition levels are written together, definition levels are written together, and finally values are written together. This means that we can use some encoding methods to store the values efficiently. Here we discuss some common encodings
Plain - we just store the plain values 
[ Apple,apple,boy,boy,donkey,apple ]


Bitpack/run length encoding 
The actual bitpack/rle algorithm is kind of complicated so we will just summarize the main idea of bitpack and run length
Bitpack - means how many bits can represent a number. For example, digits up to 6, we can use 3 bits to represent them, instead of 32 bits for an integer


Runlength - means we encode values that are back to back into 2 values, the actual value and the repetition times, for example
plain: [ apple,apple,boy,boy,donkey,apple ]
rle: [ apple2,boy2,donkey1,apple1 ]


Dictionary encoding - there are 2 variants, plain dictionary and rle dictionary
plain dictionary, this is mostly deprecated. we create a dictionary page and store the distinct values, then store the index of the dictionary value in the data pages
rle dictionary, this is the default in parquet 2.0, we further encode dictionary with rle/bitpack
Fallback - when the dictionary for encoding gets too big, Parquet just gives up using dictionary and starts writing the values using plain. So the dictionary space is preallocated and cannot grow.


Delta encoding - instead of encoding the values plainly, we encode the delta between this value and the next, this is mostly used for values that increase sequentially after being clustered


Byte split stream - the idea of byte split stream is to put the bytes into buckets based on the sequence, this may greatly improve compression later
Before:	
	ELEMENT 0      ELEMENT 1      ELEMENT 3
Bytes  AA BB CC DD    00 11 22 33    A3 B4 C5 D6 
After:
Bytes  AA 00 A3 BB 11 B4 CC 22 C5 DD 33 D6
How does parquet choose which encoding to use, well it makes the decision based on some statistics of the data as well as the datatype. You can check the full logic here:  parquet-mr/Encoding.java at master · apache/parquet-mr (github.com)
Compression methods - the 3rd layer
The above encoding methods merely applies to the column having the same value and datatype. However the file itself may still have room for compression using algorithms like snappy, gzip and ZSTD. So finally the whole file is compressed again. Here we won't elaborate more.
Speeding up queries - Dictionaries, column chunk statistics & bloom filters
There are some features of parquet which can help with speeding up queries:
Dictionary - the presence of dictionary means we can use a dictionary probe when checking whether the parquet file has the value we want. Please note that when Fallback flag is on, we may need to read the plain values to check whether the value is present
Column chunk statistics - Parquet collects basic statistics about the data at the column chunk level, which also helps with filtering
min/max - min/max can be used for the database to prune the data (such as skipping the data when its not relevant, similar to skipping indices in Clickhouse and zone map in relational database
numnulls - conveniently compute null counts
Bloom filter - while dictionary enables exact search, the storage space is big and may cause fallback to plain. Hence, Parquet also supports bloomfilters, which are efficient probabilistic data structures which may give false positive result.
Parquet in practice
In practice, we can have many ways of generating parquet files, for example using CPP, Golang etc. But in practice, it is mainly used in big data, so we shall cover its uses in Spark query engine
Parquet settings
Parquet settings: Parquet Files - Spark 3.1.1 Documentation (apache.org)
And Bloom filter settings (not found in official docs): parquet-mr/parquet-hadoop at master · apache/parquet-mr (github.com)
In normal settings, Parquet default configurations should be enough for our usage. In particular, some of the settings are not open for modification such as encoding choices and logic unless you overwrite the Parquet writer. Here are some parquet settings of interest


setting
default val
description
spark.sql.parquet.compression.codec
snappy
Compression codec to use when saving to file. This can be one of the known case-insensitive shorten names (none, uncompressed, snappy, gzip, lzo, brotli, lz4, and zstd). This will override spark.sql.parquet.compression.codec.
spark.sql.parquet.aggregatePushdown
false
If true, aggregates will be pushed down to Parquet for optimization. Support MIN, MAX and COUNT as aggregate expression. For MIN/MAX, support boolean, integer, float and date type. For COUNT, support all data types. If statistics is missing from any Parquet file footer, exception would be thrown.
only in Spark 3.2 and above
parquet.bloom.filter.enabled
false
only in Spark 3.2 and above



How to write parquet files
SparkSQL/PrestoSQL - we write SparkSQL/PrestoSQL into a table, it will generate many files with a compression method like ZSTD or Snappy, the number of files generated depends on the number of partitions in the final step, in Spark you can control this by repartitioning in the final round or using some config to control
create table xxx (
field1 string,
field2 string
) stored as parquet

insert into xxx
select * from yyy

Writing directly 
#Pyspark
df.write.format("parquet").save("./wttest2/date=2022-04-09")


How to read parquet files
SparkSQL/PrestoSQL - Build a  external table on top of the directory of parquet files and set the table data type as Parquet, it will read the underlying Parquet file and perform schema reconcilliation (match the parquet schema with Hive schema)
create table xxx (
field1 string,
field2 string
) stored as parquet

Read the files directly in Spark 
#Pyspark
df = spark.read.parquet('xxxxx.zstd.parquet')


How to analyse parquet files
Parquet CLI: parquet-mr/parquet-cli at master · apache/parquet-mr (github.com)
Using Parquet CLI, we can check the metadata of a parquet file and get a result such as this:
File path:  /Users/wentat.woong/compact-06999-8c7e5bc0-fa44-4ba6-b273-1c13f811665a.c000.zstd.parquet

```
Created by: parquet-mr version 1.10.1 (build a89df8f9932b6ef6633d06069e50c9b7970bebd1)

Properties:

                   org.apache.spark.version: 3.1.2

  org.apache.spark.sql.parquet.row.metadata: {"type":"struct","fields":[{"name":"request_id","type":"string","nullable":true,"metadata":{}},{"name":"event_timestamp","type":"long","nullable":true,"metadata":{}},{"name":"user_id","type":"long","nullable":true,"metadata":{}},{"name":"shopee_df_plain","type":"string","nullable":true,"metadata":{}},{"name":"client_ip","type":"string","nullable":true,"metadata":{}},{"name":"ip_country","type":"string","nullable":true,"metadata":{}},{"name":"domain_country","type":"string","nullable":true,"metadata":{}},{"name":"referer","type":"string","nullable":true,"metadata":{}},{"name":"user_agent","type":"string","nullable":true,"metadata":{}},{"name":"platform_id","type":"integer","nullable":true,"metadata":{}},{"name":"client_url","type":"string","nullable":true,"metadata":{}},{"name":"attributes","type":"string","nullable":true,"metadata":{}},{"name":"hit_strategy_info","type":"string","nullable":true,"metadata":{}},{"name":"action_id","type":"integer","nullable":true,"metadata":{}},{"name":"error_code","type":"integer","nullable":true,"metadata":{}},{"name":"response_extra_info","type":"string","nullable":true,"metadata":{}},{"name":"event_id","type":"integer","nullable":true,"metadata":{}},{"name":"event_name","type":"string","nullable":true,"metadata":{}},{"name":"source_system_name","type":"string","nullable":false,"metadata":{}},{"name":"event_local_datetime","type":"string","nullable":true,"metadata":{}},{"name":"event_regional_datetime","type":"string","nullable":true,"metadata":{}},{"name":"load_datetime","type":"string","nullable":false,"metadata":{}},{"name":"is_strategy_hit","type":"integer","nullable":false,"metadata":{}},{"name":"platform_desc","type":"string","nullable":true,"metadata":{}},{"name":"action_type_desc","type":"string","nullable":true,"metadata":{}},{"name":"is_ip_proxy","type":"boolean","nullable":true,"metadata":{}},{"name":"cookie","type":"string","nullable":true,"metadata":{}},{"name":"cookie_length","type":"integer","nullable":true,"metadata":{}},{"name":"ip_type_id","type":"integer","nullable":true,"metadata":{}},{"name":"tls_fp_type_id","type":"integer","nullable":true,"metadata":{}},{"name":"tls_fp_md5_hash","type":"string","nullable":true,"metadata":{}},{"name":"hit_strategy_info_v2","type":"string","nullable":true,"metadata":{}},{"name":"waf_event_id","type":"long","nullable":true,"metadata":{}},{"name":"tss_hit_module","type":"integer","nullable":true,"metadata":{}},{"name":"tss_hit_module_desc","type":"string","nullable":true,"metadata":{}},{"name":"domain","type":"string","nullable":true,"metadata":{}},{"name":"api_path","type":"string","nullable":true,"metadata":{}},{"name":"http_method","type":"string","nullable":true,"metadata":{}},{"name":"content_length","type":"string","nullable":true,"metadata":{}},{"name":"ua_type_id","type":"integer","nullable":true,"metadata":{}},{"name":"device_id","type":"string","nullable":true,"metadata":{}},{"name":"tls_fp","type":"string","nullable":true,"metadata":{}},{"name":"request_header","type":"string","nullable":true,"metadata":{}},{"name":"request_body","type":"string","nullable":true,"metadata":{}},{"name":"session_id","type":"string","nullable":true,"metadata":{}},{"name":"sap","type":"string","nullable":true,"metadata":{}},{"name":"encrypted_security_data","type":"string","nullable":true,"metadata":{}},{"name":"dfp_token","type":"string","nullable":true,"metadata":{}},{"name":"ac_hit_strategy_ids","type":"string","nullable":true,"metadata":{}},{"name":"query","type":"string","nullable":true,"metadata":{}},{"name":"query_pair","type":"string","nullable":true,"metadata":{}},{"name":"hit_strategy_ids","type":{"type":"array","elementType":"long","containsNull":true},"nullable":true,"metadata":{}},{"name":"ip_info","type":"string","nullable":true,"metadata":{}},{"name":"android_sdk_env_info","type":"string","nullable":true,"metadata":{}},{"name":"ios_sdk_env_info","type":"string","nullable":true,"metadata":{}},{"name":"web_sdk_env_info","type":"string","nullable":true,"metadata":{}},{"name":"client_id","type":"string","nullable":true,"metadata":{}},{"name":"is_support_captcha","type":"boolean","nullable":true,"metadata":{}},{"name":"experimental_event_id","type":"long","nullable":true,"metadata":{}},{"name":"experimental_waf_event_id","type":"long","nullable":true,"metadata":{}},{"name":"experimental_hit_strategy_info","type":"string","nullable":true,"metadata":{}},{"name":"tss_experimental_hit_module","type":"integer","nullable":true,"metadata":{}}]}

Schema:

message spark_schema {
  optional binary request_id (STRING);
  optional int64 event_timestamp;
  optional int64 user_id;
  optional binary shopee_df_plain (STRING);
  optional binary client_ip (STRING);
  optional binary ip_country (STRING);
  optional binary domain_country (STRING);
  optional binary referer (STRING);
  optional binary user_agent (STRING);
  optional int32 platform_id;
  optional binary client_url (STRING);
  optional binary attributes (STRING);
  optional binary hit_strategy_info (STRING);
  optional int32 action_id;
  optional int32 error_code;
  optional binary response_extra_info (STRING);
  optional int32 event_id;
  optional binary event_name (STRING);
  required binary source_system_name (STRING);
  optional binary event_local_datetime (STRING);
  optional binary event_regional_datetime (STRING);
  required binary load_datetime (STRING);
  required int32 is_strategy_hit;
  optional binary platform_desc (STRING);
  optional binary action_type_desc (STRING);
  optional boolean is_ip_proxy;
  optional binary cookie (STRING);
  optional int32 cookie_length;
  optional int32 ip_type_id;
  optional int32 tls_fp_type_id;
  optional binary tls_fp_md5_hash (STRING);
  optional binary hit_strategy_info_v2 (STRING);
  optional int64 waf_event_id;
  optional int32 tss_hit_module;
  optional binary tss_hit_module_desc (STRING);
  optional binary domain (STRING);
  optional binary api_path (STRING);
  optional binary http_method (STRING);
  optional binary content_length (STRING);
  optional int32 ua_type_id;
  optional binary device_id (STRING);
  optional binary tls_fp (STRING);
  optional binary request_header (STRING);
  optional binary request_body (STRING);
  optional binary session_id (STRING);
  optional binary sap (STRING);
  optional binary encrypted_security_data (STRING);
  optional binary dfp_token (STRING);
  optional binary ac_hit_strategy_ids (STRING);
  optional binary query (STRING);
  optional binary query_pair (STRING);
  optional group hit_strategy_ids (LIST) {
    repeated group list {
      optional int64 element;
    }
  }
  optional binary ip_info (STRING);
  optional binary android_sdk_env_info (STRING);
  optional binary ios_sdk_env_info (STRING);
  optional binary web_sdk_env_info (STRING);
  optional binary client_id (STRING);
  optional boolean is_support_captcha;
  optional int64 experimental_event_id;
  optional int64 experimental_waf_event_id;
  optional binary experimental_hit_strategy_info (STRING);
  optional int32 tss_experimental_hit_module;
}

Row group 0:  count: 98787  1.127 kB records  start: 4  total(compressed): 108.725 MB total(uncompressed):108.725 MB 

--------------------------------------------------------------------------------

                                type      encodings count     avg size   nulls   min / max

request_id                      BINARY    Z   _     98787     20.57 B    0       "00025f52-f592-427f-ac65-c..." / "ffffe14e-a3da-4596-9996-f..."

event_timestamp                 INT64     Z _ R     98787     3.41 B     0       "1679414400" / "1679500791"

user_id                         INT64     Z   _     98787     0.78 B     84129   "10033" / "1200175406"

shopee_df_plain                 BINARY    Z _ R     98787     5.37 B     73579   "" / "zykwsyB2z5LWFg7hDqJcJV6GR..."

client_ip                       BINARY    Z _ R     98787     2.93 B     0       "1.0.239.175" / "99.39.155.249"

ip_country                      BINARY    Z _ R     98787     0.46 B     0       "" / "ZA"

domain_country                  BINARY    Z _ R     98787     0.00 B     0       "SG" / "SG"

referer                         BINARY    Z _ R_ F  98787     12.10 B    0       "" / "top1sg"

user_agent                      BINARY    Z _ R_ F  98787     4.14 B     0       "" / "unirest-php/2.0"

platform_id                     INT32     Z _ R     98787     0.27 B     0       "0" / "8"

client_url                      BINARY    Z _ R_ F  98787     51.06 B    0       "http://c-api-subaccount.s..." / "https://wfm.ssc.shopee.co..."

attributes                      BINARY    Z   _     98787     644.20 B           

hit_strategy_info               BINARY    Z   _     98787     0.00 B             

action_id                       INT32     Z _ R     98787     0.05 B     0       "0" / "5"

error_code                      INT32     Z _ R     98787     0.00 B     0       "0" / "0"

response_extra_info             BINARY    Z _ R     98787     0.04 B     0       "{"challenge_type":1,"capt..." / "{}"

event_id                        INT32     Z _ R     98787     0.27 B     0       "0" / "9714"

event_name                      BINARY    Z _ R     98787     0.08 B     93108   "discovery_landing" / "search_event"

source_system_name              BINARY    Z _ R     98787     0.00 B     0       "anti_crawler" / "anti_crawler"

event_local_datetime            BINARY    Z _ R_ F  98787     4.18 B     0       "2023-03-22 00:00:00" / "2023-03-22 23:59:51"

event_regional_datetime         BINARY    Z _ R_ F  98787     4.18 B     0       "2023-03-22 00:00:00" / "2023-03-22 23:59:51"

load_datetime                   BINARY    Z _ R     98787     0.00 B     0       "2023-03-23 02:02:35.237" / "2023-03-23 02:02:35.237"

is_strategy_hit                 INT32     Z _ R     98787     0.07 B     0       "0" / "1"

platform_desc                   BINARY    Z _ R     98787     0.27 B     0       "android_app" / "unknown"

action_type_desc                BINARY    Z _ R     98787     0.05 B     616     "allow" / "force_login"

is_ip_proxy                     BOOLEAN   Z   _     98787     0.09 B     89739   "false" / "true"

cookie                          BINARY    Z _ R_ F  98787     93.63 B    0       "" / "{}"

cookie_length                   INT32     Z _ R     98787     0.67 B     0       "0" / "4305"

ip_type_id                      INT32     Z _ R     98787     0.03 B     0       "0" / "2"

tls_fp_type_id                  INT32     Z _ R     98787     0.22 B     0       "0" / "5"

tls_fp_md5_hash                 BINARY    Z _ R     98787     1.79 B     0       "" / "fffcc29122e024fe3cb419afc..."

hit_strategy_info_v2            BINARY    Z _ R_ F  98787     2.01 B     0       "[]" / "[{"strategy_id":971400000..."

waf_event_id                    INT64     Z _ R     98787     0.20 B     0       "0" / "9716"

tss_hit_module                  INT32     Z _ R     98787     0.06 B     0       "0" / "3"

tss_hit_module_desc             BINARY    Z _ R     98787     0.06 B     0       "anti_crawler" / "waf"

domain                          BINARY    Z _ R     98787     0.31 B     0       "ats.workatsea.com" / "wfm.ssc.shopee.com"

api_path                        BINARY    Z _ R     98787     1.34 B     0       "/" / "/zenkosuperfoods"

http_method                     BINARY    Z _ R     98787     0.18 B     0       "GET" / "PUT"

content_length                  BINARY    Z   _     98787     0.00 B             

ua_type_id                      INT32     Z   _     98787     0.00 B     98787   

device_id                       BINARY    Z _ R     98787     0.24 B     96651   "1399998037467459258" / "4444444444444444444"

tls_fp                          BINARY    Z _ R     98787     2.12 B     0       "" / "771,61-53-49192-49191-491..."

request_header                  BINARY    Z _ R_ F  98787     80.99 B    0       "{"1":"Content-Type:applic..." / "{"x-whs-id":"TWYW","accep..."

request_body                    BINARY    Z _ R_ F  98787     31.43 B    0       "" / "{}"

session_id                      BINARY    Z   _     98787     0.00 B             

sap                             BINARY    Z _ R_ F  98787     9.67 B     0       "{"method":"GET","region":..." / "{"signature":"zpyBEkC0Q5-..."

encrypted_security_data         BINARY    Z _ R_ F  98787     26.68 B    83402   "" / "y"

dfp_token                       BINARY    Z   _     98787     0.00 B             

ac_hit_strategy_ids             BINARY    Z _ R     98787     1.42 B     0       "GET_ats.workatsea.com_/at..." / "PUT_help.shopee.sg_/api/i..."

query                           BINARY    Z _ R_ F  98787     45.23 B    0       "" / "zarsrc=31&utm_source=zalo..."

query_pair                      BINARY    Z _ R_ F  98787     48.70 B    0       "{"CSRF_TOKEN":"b910ca37-e..." / "{}"

hit_strategy_ids.list.element   INT64     Z _ R     110597    0.25 B     92280   "7000" / "97140000003"

ip_info                         BINARY    Z   _     98787     35.50 B    0       "{"asn":"-","city":"-","is..." / "{"usagetype":"SES","threa..."

android_sdk_env_info            BINARY    Z   _     98787     8.25 B     93520   "{"android_checksum_check"..." / "{"wifi_ssid":"\"rayerayeo..."

ios_sdk_env_info                BINARY    Z   _     98787     3.71 B     95001   "{"api_hooked_info":"","ba..." / "{"wifi_ssid":"","wifi_ip_..."

web_sdk_env_info                BINARY    Z _ R_ F  98787     4.63 B     96057   "{"audio_index":"","beauti..." / "{"window_top_position":84..."

client_id                       BINARY    Z _ R     98787     0.06 B     93975   "" / ""

is_support_captcha              BOOLEAN   Z   _     98787     0.08 B     92209   "false" / "true"

experimental_event_id           INT64     Z _ R     98787     0.00 B     0       "0" / "0"

experimental_waf_event_id       INT64     Z _ R     98787     0.00 B     0       "0" / "0"

experimental_hit_strategy_info  BINARY    Z _ R     98787     0.00 B     0       "[]" / "[]"

tss_experimental_hit_module     INT32     Z   _     98787     0.00 B     98787   

Row group 1:  count: 95636  1.078 kB records  start: 114006788  total(compressed): 100.690 MB total(uncompressed):100.690 MB 

--------------------------------------------------------------------------------

                                type      encodings count     avg size   nulls   min / max

request_id                      BINARY    Z   _     95636     20.58 B    0       "0002b378-c388-42bd-acd2-9..." / "fffc9037-04d4-4da6-890c-3..."

event_timestamp                 INT64     Z _ R     95636     3.37 B     0       "1679414401" / "1679500798"

user_id                         INT64     Z   _     95636     0.67 B     83711   "11742" / "973157000"

shopee_df_plain                 BINARY    Z _ R     95636     4.72 B     74195   "" / "zzgSI7P7xIljQVmv2JAR7B03a..."

client_ip                       BINARY    Z _ R     95636     2.82 B     0       "1.10.202.98" / "99.246.22.202"

ip_country                      BINARY    Z _ R     95636     0.45 B     0       "" / "ZA"

domain_country                  BINARY    Z _ R     95636     0.00 B     0       "SG" / "SG"

referer                         BINARY    Z _ R_ F  95636     12.25 B    0       "" / "https://zimpolo.com/"

user_agent                      BINARY    Z _ R_ F  95636     3.96 B     0       "" / "unirest-php/2.0"

platform_id                     INT32     Z _ R     95636     0.27 B     0       "0" / "8"

client_url                      BINARY    Z _ R_ F  95636     52.20 B    0       "http://c-api-subaccount.s..." / "https://wfm.ssc.shopee.co..."

attributes                      BINARY    Z   _     95636     617.98 B           

hit_strategy_info               BINARY    Z   _     95636     0.00 B             

action_id                       INT32     Z _ R     95636     0.05 B     0       "0" / "5"

error_code                      INT32     Z _ R     95636     0.00 B     0       "0" / "0"

response_extra_info             BINARY    Z _ R     95636     0.04 B     0       "{"challenge_type":1,"capt..." / "{}"

event_id                        INT32     Z _ R     95636     0.26 B     0       "0" / "9714"

event_name                      BINARY    Z _ R     95636     0.08 B     90601   "discovery_landing" / "spx_event"

source_system_name              BINARY    Z _ R     95636     0.00 B     0       "anti_crawler" / "anti_crawler"

event_local_datetime            BINARY    Z _ R_ F  95636     4.10 B     0       "2023-03-22 00:00:01" / "2023-03-22 23:59:58"

event_regional_datetime         BINARY    Z _ R_ F  95636     4.10 B     0       "2023-03-22 00:00:01" / "2023-03-22 23:59:58"

load_datetime                   BINARY    Z _ R     95636     0.00 B     0       "2023-03-23 02:02:35.237" / "2023-03-23 02:02:35.237"

is_strategy_hit                 INT32     Z _ R     95636     0.07 B     0       "0" / "1"

platform_desc                   BINARY    Z _ R     95636     0.27 B     0       "android_app" / "unknown"

action_type_desc                BINARY    Z _ R     95636     0.05 B     560     "allow" / "force_login"

is_ip_proxy                     BOOLEAN   Z   _     95636     0.08 B     88200   "false" / "true"

cookie                          BINARY    Z _ R_ F  95636     79.81 B    0       "" / "{}"

cookie_length                   INT32     Z _ R     95636     0.63 B     0       "0" / "3688"

ip_type_id                      INT32     Z _ R     95636     0.03 B     0       "0" / "2"

tls_fp_type_id                  INT32     Z _ R     95636     0.23 B     0       "0" / "5"

tls_fp_md5_hash                 BINARY    Z _ R     95636     1.70 B     0       "" / "fffd2ec209e0d89e606cbb061..."

hit_strategy_info_v2            BINARY    Z _ R_ F  95636     1.93 B     0       "[]" / "[{"strategy_id":971400000..."

waf_event_id                    INT64     Z _ R     95636     0.20 B     0       "0" / "9716"

tss_hit_module                  INT32     Z _ R     95636     0.06 B     0       "0" / "3"

tss_hit_module_desc             BINARY    Z _ R     95636     0.06 B     0       "anti_crawler" / "waf"

domain                          BINARY    Z _ R     95636     0.29 B     0       "ats.workatsea.com" / "wfm.ssc.shopee.com"

api_path                        BINARY    Z _ R     95636     1.33 B     0       "/" / "/zenkosuperfoods"

http_method                     BINARY    Z _ R     95636     0.18 B     0       "GET" / "PUT"

content_length                  BINARY    Z   _     95636     0.00 B             

ua_type_id                      INT32     Z   _     95636     0.00 B     95636   

device_id                       BINARY    Z _ R     95636     0.21 B     93875   "1399683719822439175" / "1638570858845819429"

tls_fp                          BINARY    Z _ R     95636     2.05 B     0       "" / "771,61-53-49192-49191-491..."

request_header                  BINARY    Z _ R_ F  95636     76.18 B    0       "{"0":"Accept:*/*","host":..." / "{"x-whs-id":"TWYW","conte..."

request_body                    BINARY    Z _ R_ F  95636     30.45 B    0       "" / "{}"

session_id                      BINARY    Z   _     95636     0.00 B             

sap                             BINARY    Z _ R_ F  95636     8.72 B     0       "{"method":"GET","region":..." / "{"signature":"zxkNMfBsS28..."

encrypted_security_data         BINARY    Z _ R_ F  95636     22.94 B    82594   "" / "z"

dfp_token                       BINARY    Z   _     95636     0.00 B             

ac_hit_strategy_ids             BINARY    Z _ R     95636     1.40 B     0       "GET_ats.workatsea.com_/at..." / "PUT_help.shopee.sg_/api/i..."

query                           BINARY    Z _ R_ F  95636     46.50 B    0       "" / "zarsrc=31&utm_source=zalo..."

query_pair                      BINARY    Z _ R_ F  95636     49.92 B    0       "{"CSRF_TOKEN":"47f3be6f-4..." / "{}"

hit_strategy_ids.list.element   INT64     Z _ R     106654    0.24 B     89741   "7000" / "97140000003"

ip_info                         BINARY    Z   _     95636     35.36 B    0       "{"asn":"-","city":"-","do..." / "{"usagetype":"SES","threa..."

android_sdk_env_info            BINARY    Z   _     95636     7.02 B     91310   "{"android_checksum_check"..." / "{"wifi_ssid":"\"winter so..."

ios_sdk_env_info                BINARY    Z   _     95636     3.22 B     92519   "{"api_hooked_info":"","ba..." / "{"wifi_ssid":"","wifi_ip_..."

web_sdk_env_info                BINARY    Z _ R_ F  95636     4.76 B     92982   "{"audio_index":"","batter..." / "{"window_top_position":84..."

client_id                       BINARY    Z _ R     95636     0.06 B     91471   "" / ""

is_support_captcha              BOOLEAN   Z   _     95636     0.08 B     89801   "false" / "true"

experimental_event_id           INT64     Z _ R     95636     0.00 B     0       "0" / "0"

experimental_waf_event_id       INT64     Z _ R     95636     0.00 B     0       "0" / "0"

experimental_hit_strategy_info  BINARY    Z _ R     95636     0.00 B     0       "[]" / "[]"

tss_experimental_hit_module     INT32     Z   _     95636     0.00 B     95636  
```

Encodings
([_SGLB$Z?])\s([_RD?\s])?\s(R)?(_)?(D)?(\sF)?

First group, compression codec
_ = uncompressed
S = snappy
G = gzip
L = LZ0
B = Brotli
4 = LZ4
Z = ZSTD
? = unknown
Second group, enum of dictionary encoding
_ = plain
R= RLE
D = delta byte array
? = unknown
\s =  no dictionary
Third group, whether encodings has RLE
R = data encodings has RLE
Forth group, whether data encodings has plain
 _ = data encodings has plain
Fifth group, whether data encodings has delta
D =  data encodings has delta
Sixth group, whether data has fallback
\sF = data has fallback
In our example, we can understand it as follows:
Z   _ = ZSTD compressed, no dictionary, data encoded in plain, example: request_id
Z _ R = ZSTD compressed, data encoded in plain dictionary, but data is RLE example: event_timestamp
Z _ R_ F  = ZSTD compressed, dictionary encoded in plain, data encoded in RLE and plain, there is a fallback, example: referer, user_agent
Key takeaways
Theory - theory of parquet is useful not just for knowledge's sake but is widely used in systems such as Spark, Hudi and Flink. Even new data formats such as delta and arrow are inspired by parquet. Understanding parquet helps to understand the more complicated systems and data formats, and hopefully lets you design better systems 
Query - parquet stores the columns separately, so by querying only columns we need, we can speed up the queries by only reading the related column chunks
Schema design - if you map the schema directly to parquet types, you can save the key for space and also use more efficient data types and encodings. So its definitely more space efficient to store the schema as detailed as it can be, instead of using a big string JSON field for example
Data colocation - due to the RLE/bitpack encoding method used in parquet, if we skew the data, we can efficiently store it. So if we put the same data in the same parquet file, we can save a lot of storage space, in fact, there is an ongoing effort by our team to improve the storage space usage using this method 
