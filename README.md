# Realtime Aggregator

## Introduction
Hundreds of transit authorities worldwide freely distribute realtime
 information about their trains, buses and other services
using a variety of formats, principally the
General Transit Feed Specification (GTFS) format. 
This realtime data is primarily designed to help their customers plan
their journeys in the moment, however when aggregated over a period of
 time it becomes an extensive data set that 
	can be used to answer interesting questions.
Such data could be used to evaluate past performance ('what percentage
of trains arrived on time at Penn Station between 5pm and 7pm?')
or possibly to improve the transit authority's realtime predictions.

This software was developed in order to aggregate such realtime data.
The software was designed with the following principles in mind:
* **Be reliable**: the program is tasked with generating a *complete* data
set: the aggregation process
	cutting out for an hour is unacceptable.
	The software is designed to be robust and to be easily deployed
	 with multiple layers of redundancy.
 
* **Be space efficient**: the flip side of redundancy is that significantly
	more data is downloaded and processed than needed.
	If the New York City subway realtime data was downloaded every 5
	seconds, 2 gigabytes of data would be generated each day!
	The software removes duplicate and corrupt
	data, compresses data by the hour,
	and offers the facility of transferring data from
	the local (expensive) server to remote (cheap) object storage.

* **Be flexible**: transit authorities don't just use GTFS: the New York
	City transit authority, for example,
	also distributes data in an ad hoc XML format.
	The software can handle any kind of file-based realtime feed once 
	the user provides a Python function for determining
	the feed's publication time from its content.
	The software has GTFS functionality built-in.

## Getting Started

### Prerequisites

The software is written in Python 3. It requires the package `requests` 
to perform the downloads; this can be installed, for example, using `pip`:

```
$ pip3 install requests
```

A number of optional features require additional Python 3 packages. 
To use the built-in GTFS functionality, the `gtfs-realtime-bindings` package
is required:

```
$ pip3 install gtfs-realtime-bindings
```

The software can transfer compressed data files from the local server to a 
remote object storage server, for example
	[Amazon S3 storage](https://aws.amazon.com/s3/) or 
	[Digital Ocean spaces](https://www.digitalocean.com/products/object-storage/).
To use this functionality, the `boto3` package is required:
```
$ pip3 install boto3
```

(You will additionally need to specify your object storage settings 
in the `remote_settings.py` file.
The required settings are described in detail in that file.)

Given these package dependencies, you may consider deploying the software in a 
Python virtual environment. The full set of dependencies can be installed
using the `requirements.txt` file in a virtual environment in the usual way:
```
(env) $ pip install -r requirements.txt
```


### Installing

The program files can be placed anywhere; the only requirement is that
	the software must have permission to create and delete directories
	 and files within its subdirectory.
(The program can be configured to operate in a different directory, 
	so that it won't require read and write
	permissions in the directory it is installed; see the 
	[advanced usage guide](docs/advanced_usage.md) for details.)


Before running the program, you need to specify the realtime feeds that 
you wish to aggregate in `remote_settings.py`.
For each feed you wish to aggregate you will need to provide four items:
1. A unique identifier `uid` for the feed. This is merely used for the 
	program to internally distinguish the different feeds you are
	aggregating, and is completely up to you. 
1. The URL where the feed is to be downloaded from.
1. A file extension for the feed, for example `gtfs`, `txt`, or `xml`.
1. A Python 3 function that, given the location of a feed download locally,
	determines if it is a valid feed (for example,
	was not corrupted during the download process) and, if so, returns
	the time at which the feed was published by the transit authority.
	Such a function for GTFS Realtime is provided in the default
	 `remote_settings.py`.

Feed settings are set in `remote_settings.py`, and more detailed instructions
are provided in that file.
For convenience, the program is distributed with the feed settings for two
New York City subway lines,
	 although you will need an 
	[API key](http://datamine.mta.info/) from the 
	Metropolitan Transit Authority 
	to use them.

The program is invoked through a command line interface.
The full interface can be explored by running:
```
$ python3 realtimeaggregator.py -h
```
To begin, you're going to want to test the software to ensure that it is
running correctly and downloading your feeds correctly.
To do this, run:
```
$ python3 realtimeaggregator.py test
```
This command performs all the tasks involved in the aggregating process,
as described below.
If you have specified remote object storage, `.tar.bz2` files containing the
feeds will be uploaded to your remote storage.
You should check your remote storage to ensure the files were uploaded 
successfully. 
By default, the object key for the aggregated files for feed `uid` aggregated 
in clock hour `hh` on the date `mm/dd/yyyy` will be
```
realtime-aggregator/yyyy-mm-dd/hh/uid-yyyy-mm-ddThh.tar.bz2
```

Otherwise, the `.tar.bz2` files may be found and inspected locally in the 
`store/compressed/` subdirectory.
The aggregated files for feed `uid` aggregated in clock hour `hh` on the
date `mm/dd/yyyy` will be
```
store/compressed/yyyy-mm-dd/hh/uid-yyyy-mm-ddThh.tar.bz2
```

The string `yyyy-mm-ddThh` appearing in the file name is
	 a [UTC 8610](https://en.wikipedia.org/wiki/ISO_8601)
representation of the clock hour.
As you can see, by default data for different days and hours is placed
in different directories,
however the file naming scheme is designed so that all the data you
aggregate can subsequently be placed together in a single directory.



### Deploying

After using the test feature to verify that the software is working
and that your settings are valid,
	you will want to deploy the software to begin the aggregation
proper.
To aggregate 24/7, the software should be running on a server that is always on!

The aggregation process involves three *tasks*, with an optional fourth task.
In order to aggregate, these tasks need to be scheduled regularly by Cron
or a similar facility.
The tasks are:

1. **Download task**.
	This is the task that actually downloads the feeds from the 
	transit authority's server to your local server.
	It is the only task that runs continuously.
	It downloads the feeds at a certain frequency (by default every
	14 seconds) and concludes after a certain amount of time 
	(by default after 15 minutes).
	Your system should be set up so that when a download task concludes,
	Cron starts a new download task to keep the download process going.

	The download task is the most critical component of the software.
	To create a complete data set, it is essential that there is at
	least one download task running at all times.
	In deployments, one should consider scheduling download tasks 
	with redundancy.
	For example, one could schedule a download task of duration 15	
	minutes to start every 5 minutes.
	That way, at a given time three download tasks will be running
	 simultaneously and so up to two can fail without any data loss.

2. **Filter task**.
	This task filters the files that have been downloaded by
	 removing duplicates and corrupt files.
	It can be run as frequently as one wishes: by default 
	it runs every 5 minutes.

3. **Compress task**.
	This task compresses the filtered feed downloads for a given clock
	hour into one `.tar.bz2` archive for each feed.
	The compress task only compresses a given clock hour when the program
	knows that all the downloads for that clock hour have been filtered.
	(However, if more downloads for a given clock hour subsequently
	appear, the compress task will add these to the relevant archive.)
	Because the compress task compressess by the clock hour, it 
	need only be scheduled at most once an hour.

4. **Archive task**.
	This task trasfers the compressed archives from the local server
	to remote object storage.
	This is esentially a money-saving operation, as bucket storage is
	about 10% the cost of server space per gigabyte.


The `schedules.crontab` file contains Cron instructions for 
scheduling the tasks as described here.
This file needs to be installed for Cron to work from it:
```
$ crontab schedules.crontab
```
Remember that usually each user only gets one crontab file.
If you
have another crontab files in use, you will need to merge the 
two files together before invoking `crontab`.

Once the Cron file has been installed, the aggregation will begin in the
 background.
To ensure the aggregation is running successfully, you should check
your object storage or local server to see
that the relevant `.tar.bz2` files are appearing and that they contain
the correct feeds and at the right frequency.
Note that after you install the Cron file, it will take at least an hour
for these archives to appear.
You should also consult the log files, which describe how successful the
program is in terms of number of files downloaded,
number of compressed archives created, etc. 
The [reading the logs guide](docs/reading_the_logs.md) describes how you
can navigate the log files.



### Two notes on consistent aggregation

As mentioned before, it is essential that the software be downloading 
feeds all the time.
Redundancy may be introduced by scheduling multiple, overlapping download
 tasks.
One can introduce further redundancy by scheduling multiple, autonomous
 aggregator sessions using the Cron file.
Such sessions would track the same feeds, but download to different 
directories locally, and then, when uploading to remote storage,
	use different object keys to store the output simultaneously.
See the [advanced usage guide](docs/advanced_usage.md).

You will be running the software on a server, but sometimes it may be
necessary to restart the server or otherwise pause the aggregation
on that box.
In this case, one can run the aggregation software with the same object
storage settings on a different device. 
The software is designed so that the compressed archive files from two
different instances of the program
	 being uploaded to the same location in the object storage 
	will be merged (rather than one upload overwritting the other).
However this is a little bit delicate to get right in practice; see
 the [advanced usage guide](docs/advanced_usage.md).




## What next?


The `docs` directory contains further documentation that may be of interest.

* The [reading the logs guide](docs/reading_the_logs.md) describes how
	 you may navigate the log files
	to ensure the aggregation is operating succesfully.

* The [advanced usage guide](docs/advanced_usage.md) gives instructions 
	on going beyond the basic aggregation
	discussed here.

* The [developers guide](docs/developers_guide.md) describes the software 
	for those interested in knowing
	how it works internally, and how it may be changed.









