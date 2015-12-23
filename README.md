


#Job configurations#
This entails the configurations for the jobs that will be executed, which is independent from
the system configuration. This should be a JSON file and either should be located at the
root of your app at /job.json or passed in via the commandline at startup using the flag -j 'path/to/job'.

Example Job
```
{
    "name": "Reindex Events",
    "lifecycle": "once",
    "analytics": false,
    "operations": [
        {
            "_op": "elasticsearch_reader",
            "index": "events-*",
            "size": 5000,
            "start": "2015-07-08",
            "end": "2015-07-09",
            "interval": "5_mins",
            "date_field_name": "created",
            "full_response": true
        },
        {
            "_op": "elasticsearch_index_selector",
            "index": "bigdata3",
            "type": "events",
            "indexPrefix": "events",
            "timeseries": "daily",
            "date_field_name": "created"
        },
        {
            "_op": "elasticsearch_bulk_insert",
            "size": 5000
        }
    ]
}
```

Job level configuration options

| Configuration | Description | Type |  Notes
|:---------: | :--------: | :------: | :------:
name | Name for the given job | String | optional
lifecycle | Determines system exiting behaviour. Set to either **"once"** which will run the job to completion then the process will exit or **"persistent"** in which the process will continue until it is shutdown manually.  | String | required
analytics | Determines if analytics should be ran for each slice. If used the native analytics will count the number of documents entering and leaving each step as well as the time it took and log it with the worker id, pid and what slice it was working on. You may specify a custom reporter to be used instead of the native analytics functionality. Please refer to the reporters section for more information. | Boolean | optional, even if you specify a reporter, this must be set to true for it to run
operations | An array containing all the operations as well as their configurations. Typically the first is the reader and the last is the sender, with as many intermediate processors as needed. | Array | required
max_retries | Number of times a given slice of data will attempt to process before continuing on | Number | optional
workers | Number of worker instances that will process data, depending on the nature of the operations you may choose to over subscribe the number of workers compared to the number of cpu's | Number | optional, defaults to 5
progressive_start | Period of time (in seconds) in which workers will evenly instantiate, not specifiying this option all workers will spin up at the same time   | Number | optional, if you have 10 workers and this option is set to 20, then a new worker will instantiate every 2 seconds

##Readers##

###elasticsearch_reader###
Used to retrieve elasticsearch data based on dates. This reader has different behaviour if lifecycle is set to "once" or "persistent"


Example configuration if lifecycle is set to "once"

```
    {
      "_op": "elasticsearch_reader",
      "index": "events-*",
      "size": 5000,
      "start": "2015-10-26T21:33:27.190Z",
      "end": ""2015-10-27T21:33:27.190Z",
      "interval": "10_mins",
      "date_field_name": "created",
      "full_response": true
    }
```
In this mode, there is a definite start and end time. Each slice will be based off of the interval and size configurations. If the number of documents exceed the size within a given interval, it will recurse and and split the interval in half until the number of documents is less than or equal to size. If this cannot be achieved then the size restraint will be bypassed and the reader will process a one millisecond interval.

Once it has reached the time specified at end and all workers have finished, the main process will finish and exit.


| Configuration | Description | Type |  Notes
|:---------: | :--------: | :------: | :------:
\_op| Name of operation, it must reflect the exact name of the file | String | required
index | Which index to read from | String | required
size | The limit to the number of docs pulled in a chunk, if the number of docs retrieved by the slicer exceeds this number, it will cause the slicer to recurse to provide a smaller batch | Number | optional, defaults to 5000
start | The start date ( as ISOstring or in ms) to which it will read from | String/Number | required, inclusive
end | The end date (ISOstring or in ms) to which it will read to| String/Number | required, exclusive
interval | The time interval in which it will read from, the number must be separated from the unit of time by an underscore. The unit of time may be months, weeks, days, hours, minutes, seconds, millesconds or their appropriate abbreviations | String | optional, default set to "5_minutes"
date_field_name | field name where the date of the document used for searching resides | String | required
full_response | If set to true, it will return the native response from elasticsearch with all meta-data included. If set to false it will return an array of the actual documents, no meta data included | Boolean | optional, defaults to false


The persistent mode expects that there is a continuous stream of new data coming into elasticsearch and that it has a date field when it was uploaded. On initializing this job, it will begin to check to see if there are documents a minute ahead of the time it initialized. Once it has found documents a minute ahead it assumes the documents for the current minute are complete and will begin processing them. If after a minute there are no documents found then the timer will increment and begin searching for documents in the next minute.

A noticeable difference in the configurations is that "start" and "end" are time ranges within that given minute. If you just have one instance of teraslice then it should be the whole minute (0 to 59). However you may use multiple instances reading a specific portion of that minute (ie 2 instances reading from 0-29, 30-59)

Example configuration if lifecycle is set to "persistent"

```
{
     "_op": "elasticsearch_reader",
     "index": "someIndex",
     "size": 5000,
     "start": "00_s",
     "end": "29_s",
     "date_field_name": "created",
     "full_response": true
}
```
Differences

| Configuration | Description | Type |  Notes
|:---------: | :--------: | :------: | :------:
start | The start time of the given minute to begin reading. Needs to be formatted like the interval configuration | String | required, example "5_s"
end | The end time (inclusive) of the given minute from which your reading. Needs to be formatted like the interval configuration | String | required, example "10_s"

###elasticsearch_data_generator###
Used to generate sample data for your elasticsearch cluster. You may use the default data generator which creates randomized data fitting the format listed below or you may create your own custom schema using the [json-schema-faker](https://github.com/json-schema-faker/json-schema-faker) package to create data to whatever schema you desire.

Default generated data :
```
{ ip: '1.12.146.136',
  userAgent: 'Mozilla/5.0 (Windows NT 5.2; WOW64; rv:8.9) Gecko/20100101 Firefox/8.9.9',
  url: 'https://gabrielle.org',
  uuid: '408433ff-9495-4d1c-b066-7f9668b168f0',
  ipv6: '8188:b9ad:d02d:d69e:5ca4:05e2:9aa5:23b0',
  location: '-25.40587, 56.56418',
  bytes: 4850020 }

```

Example configuration
```
{
    "_op": "elasticsearch_data_generator",
    "size": 25000000,
    "file_path": "some/path/to/file.js"
}
```

| Configuration | Description | Type |  Notes
|:---------: | :--------: | :------: | :------:
\_op | Name of operation, it must reflect the exact name of the file | String | required
size | If lifecycle is set to "once", then size is the total number of documents that the generator will make. If lifecycle is set to "persistent", then this generator will will constantly stream data to elasticsearch in chunks as big as the size indicated | Number | required
file_path | File path to where custom schema is located | String | optional, the schema must be exported Node style "module.exports = schema"


###file_import###
Import data read from files to elasticsearch. This will read the entire file, the data is expected to be stored as JSON.

Example configuration
```
{
    "_op": "file_import",
    "path": "some/path/"
}
```

| Configuration | Description | Type |  Notes
|:---------: | :--------: | :------: | :------:
\_op | Name of operation, it must reflect the exact name of the file | String | required
path | File path to where directory containing files to upload are located. It will recursively walk the directory and all sub-directories and gather all files to upload them | String | required


##Processors##

###elasticsearch_index_selector###
This processor formats the incoming data to prepare it for the elasticsearch bulk request. It accepts either an array of data or a full elasticsearch response with all associated meta-data. It should be noted that the resulting formatted array required for the bulk request will always double the length of the incoming array

Example configuration
```
{
     op: 'elasticsearch_index_selector',
     type: 'events',
     indexPrefix: 'events',
     timeseries: 'daily',
     date_field: '@timestamp'
}
```
| Configuration | Description | Type |  Notes
|:---------: | :--------: | :------: | :------:
\_op | Name of operation, it must reflect the exact name of the file | String | required
index | Index to where the data will be sent to, if you wish the index to be based on a timeseries use the timeseries option instead | String | optional
type | Set the type of the data for elasticsearch. If incoming data is from elasticsearch it will default to the type on the metadata if this field is not set. This field must be set for all other incoming data | String | optional
preserve_id | If incoming data if from elasticsearch, set this to true if you wish to keep the previous id else elasticsearch will generate one for you (upload performance is faster if you let it auto-generate) | Boolean | optional, defaults to false
id_field | If you wish to set the id based off another field in the doc, set the name of the field here | String | optional
timeseries | Set to either "daily", "monthly" or "yearly" if you want the index to be based off it, must be used in tandem with index_prefix and date_field | String | optional
index_prefix | Used with timeseries, adds a prefix to the date ie (index_prefix: "events-" ,timeseries: "daily => events-2015.08.20 | String | optional, required if timeseries is used
date_field | Used with timeseries, specify what field of the data should be used to calculate the timeseries | String | optional, but required if using timeseries defaults to @timestamp
update | Specify if the data should update existing records, if false it will index them | Boolean | optional, defaults to false
upsert_fields | if you are updating the documents, you can specify fields to update here (it should be an array containing all the field names you want), it defaults to sending the entire document | Array | optional, defaults to []

###file_chunker###
This prepares the data to be sent to HDFS

Example configuration
```
{
    "_op": "elasticsearch_bulk_insert",
    "size": 5000
}
```
| Configuration | Description | Type |  Notes
|:---------: | :--------: | :------: | :------:
\_op | Name of operation, it must reflect the exact name of the file | String | required
timeseries | Set to an interval to have directories named using a date field from the data records | String | optional, possible choices are 'daily', 'monthly', and 'yearly'
date_field | Which field in each data record contains the date to use for timeseries. Only useful if "timeseries" is also specified | String | optional, defaults to date
directory | Path to use when generating the file name | String | required,  defaults to "/"
filename | Filename to use. This is optional and is not recommended if the target is HDFS. If not specified a filename will be automatically chosen to reduce the occurence of concurrency issues when writing to HDFS | String | optional
chunk_size | Size of the data chunks. Specified in bytes. A new chunk will be created when this size is surpassed | Number | required, defaults to 50000

##Senders##

###elasticsearch_bulk_insert###
This sends a bulk request to elasticsearch

Example configuration
```
{
    "_op": "elasticsearch_bulk_insert",
    "size": 5000
}
```
| Configuration | Description | Type |  Notes
|:---------: | :--------: | :------: | :------:
\_op | Name of operation, it must reflect the exact name of the file | String | required
size | the maximum number of docs it will send in a given request, anything past it will be split up and sent | Number | required, not the the number should always be an even number if not then you will run into a bulk request error due to the nature of the formatting it requires for a bulk request

###file_export###
Used to export elasticsearch data in a timeseries to the filesystem. This will first create directories based on the interval specified in the elasticsearch_reader config, and store the actual file to the correct directory.

Example configuration
```
{
     "_op": "file_export",
     "path": "/path/to/export",
     "metadata": false
}
```

| Configuration | Description | Type |  Notes
|:---------: | :--------: | :------: | :------:
\_op | Name of operation, it must reflect the exact name of the file | String | required
path | path to directory where the data will be saved to, directory must pre-exist | String | required
elastic_metadata | set to true if you would like to save the metadata of the doc to file | Boolean | optional, defaults to false

###hdfs_file_append###
Used to export data to HDFS, used in tandem with file_chunker.

Example configuration
```
{
     "_op": "hdfs_file_append",
     "namenode_host": "localhost",
     "namenode_port": 50070,
     "user": "hdfs"
}
```

| Configuration | Description | Type |  Notes
|:---------: | :--------: | :------: | :------:
\_op | Name of operation, it must reflect the exact name of the file | String | required
namenode_host | Host running HDFS name node | String | required
namenode_port | Port that the HDFS name node is running on | Number | optional, defaults to 50070
user | User to use when writing the files | String | optional, defaults to hdfs

##Reporters##


