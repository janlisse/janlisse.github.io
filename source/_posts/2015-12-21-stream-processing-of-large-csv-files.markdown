---
layout: post
title: "Stream processing of large CSV files"
date: 2015-12-21 15:42:08 +0100
comments: true
categories: [Scala, Akka Streams]
---

Every now and then every backend developer faces the situation to integrate/process third party data
into the companies system, provided in the form of plain old CSV files. 
But what if the CSV file is to huge to fit into memory entirely, but you still want to leverage
parallel computing to get things done in finite time? 
Once again the excellent [Akka Streams] (http://doc.akka.io/docs/akka-stream-and-http-experimental/1.0/) framework comes to the rescue.

Processing big csv files with Akk Streams is quite straightforward. First you need a [Source] that emits ByteStrings from a local file or
 another storage location like S3. Then you can leverage Akka Streams classes to group the ByteStrings to lines. Afterwards
 you can then feed those lines into a CSV Parser and emit the parsed fields. Finally you can add your own processing logic i.e. mapping to custom
 model and sending to an API or Backend Service.
 
 With SynchronousFileSource Akka Streams already provides the ability to stream ByteStrings from a local file.
 Grouping them to lines is as simple as:
 
 ``` Scala
 
 private def toLine(implicit format: CSVFormat) = Framing.delimiter(
     ByteString(format.lineTerminator),
     maximumFrameLength = 4096,
     allowTruncation = true
   )
 
 ```
 
 The CSVFormat that is passed implicitly determines delimiters, line separators etc. It must match the one that is used in the CSV data to be parsed. 
 The default format uses "," as delimiter and "\r\n" as line separator.
 
 
 For the CSV parsing part we have written a custom Akka Streams PullPushStage that transforms a raw line into a Map[String, String] with the keys being
 field names from the CSV header line. Thus this stage requires a header line to exist in the CSV. The resulting Map is emitted together
 with the current line number in the file.
 
 Our implementation utilizes an existing [Scala CSV parsing](https://github.com/tototoshi/scala-csv) lib.
 
``` Scala

class CsvStage()(implicit format: CSVFormat) extends PushPullStage[String, (Int, Map[String, String])] {

  private val parser = new CSVParser(format)
  private var headers: Option[List[String]] = None
  private var line = 0

  override def onPush(elem: String, ctx: Context[(Int, Map[String, String])]): SyncDirective = {
    val parsed = parser.parseLine(elem)
    line = line + 1
    if (headers.isEmpty) {
      headers = parsed
      ctx.pull()
    } else if (parsed.isEmpty) {
      ctx.pull()
    } else {
      val l = line -> headers.get.zip(parsed.get).toMap
      ctx.push(l)
    }
  }

  override def onPull(ctx: Context[(Int, Map[String, String])]): SyncDirective = ctx.pull()
}


```

Note that PushPullStage access is guaranteed to be thread-safe, similar to how you would encapsulate state within an Akka Actor.
Thus you can safely use mutable state within your implementation. Inside our implementation there is no magic. 
We expect the first line to be the header line, store this and do not push it further down the stream. Beginning with the second
line we zip the headers together with the parsed fields from the CSV parser and emit it with the current line number.

Finally an example how to kickoff the processing:

``` Scala

SynchronousFileSource(csvFile).via(FlowBuilder.csvFlow).
    runWith(Sink.foreach(println)).onComplete( _ => system.shutdown())

```

You can find all the code [here](https://github.com/janlisse/csv-flow) on github.






