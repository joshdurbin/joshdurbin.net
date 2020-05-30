+++
title = "A concurrent, Groovy thread ripper"
description = "Exercise and exhaust your multi-core CPU with this simple groovy example"
date = "2018-09-24"
tags = ["groovy", "concurrency"]
+++

This was a fun bit; using [Apache Tika](https://tika.apache.org/) to detect MIME types of files on your machine
with little code and effort. The script takes a location, the root location from which to
recurse, and generates a set of absolute paths to scrutinize. It then establishes a latch equal to
the size of the set to block reporting until the analysis is complete. The script then
iterates through the absolute paths, submitting jobs to the executor for processing, which each 
hold a reference to a thread safe map and counter object use for tallying.

Throw the script a large directory and watch it run!

Tika is super useful in pre-processing steps, data pipelines, when input type guarantees
are critical. 

```groovy
#!/usr/bin/env groovy

@Grapes([
    @Grab(group='org.apache.tika', module='tika-core', version='1.18')
])

import org.apache.tika.Tika

import java.util.concurrent.ConcurrentHashMap
import java.util.concurrent.Executors
import java.util.concurrent.atomic.LongAdder
import java.util.concurrent.CountDownLatch

import static groovy.io.FileType.FILES

def cli = new CliBuilder(header: 'MIME Type Reporter', usage:'./mimeReporter <directoryToScan>', width: 100)

def cliOptions = cli.parse(args)

if (cliOptions.help || cliOptions.arguments().size() != 1) {
  cli.usage()
  System.exit(0)
}

def results = new ConcurrentHashMap()
def fileAbsolutePaths = []
def tika = new Tika()
def executor = Executors.newWorkStealingPool()

new File(cliOptions.arguments().first()).eachFileRecurse(FILES) { file ->
  fileAbsolutePaths << file.absolutePath
}

def latch = new CountDownLatch(fileAbsolutePaths.size())

println "Processing ${fileAbsolutePaths.size()} files..."

fileAbsolutePaths.each { filePath ->
  executor.submit {
    try {
      results.computeIfAbsent(tika.detect(new File(filePath)), { k -> new LongAdder() }).increment()
    } finally {
      latch.countDown()
    }
  }
}

latch.await()
executor.shutdown()

def formatOutput = '%-40s occurred %d times%n' 

results.each { contentType, counter ->
  System.out.format(formatOutput, contentType, counter.intValue())
}
```

That script run on a relatively fresh AEM 6.4 author install (with SP1), yields the following
reported MIME Types:

```
➜  ~ time ./mimeReporter aem_6.4_author
Processing 6145 files...
application/xml                          occurred 29 times
application/x-sh                         occurred 6 times
image/jpeg                               occurred 506 times
text/x-log                               occurred 12 times
image/gif                                occurred 1 times
image/svg+xml                            occurred 13 times
application/x-archive                    occurred 1 times
application/java-vm                      occurred 208 times
application/x-shockwave-flash            occurred 1 times
application/x-tika-msoffice              occurred 1 times
application/x-msdownload; format=pe32    occurred 2 times
application/java-archive                 occurred 791 times
application/vnd.apple.keynote            occurred 1 times
text/x-matlab                            occurred 2 times
application/javascript                   occurred 68 times
application/gzip                         occurred 70 times
text/x-jsp                               occurred 69 times
text/x-java-source                       occurred 152 times
application/x-dosexec                    occurred 3 times
text/plain                               occurred 2886 times
text/x-java-properties                   occurred 3 times
image/png                                occurred 535 times
application/x-bat                        occurred 6 times
application/octet-stream                 occurred 256 times
application/pdf                          occurred 1 times
application/msword                       occurred 5 times
application/json                         occurred 2 times
application/java-serialized-object       occurred 1 times
audio/mpeg                               occurred 28 times
application/x-tar                        occurred 3 times
text/html                                occurred 10 times
image/tiff                               occurred 29 times
application/x-font-ttf                   occurred 15 times
application/zip                          occurred 375 times
text/css                                 occurred 53 times
video/mp4                                occurred 1 times
./mimeReporter aem_6.4_author  16.48s user 1.73s system 565% cpu 3.221 total
```
