## Hadoop hello world: counting words
In this chapter, we will add functionality step-by-step to get to a
minimal working hadoop example: a program that, given a set of input files, counts
frequency of word occurences.
### Parsing command line arguments
First of, let's add a proper argument parses so that we can pass arguments like 
`--input input.txt` and get some argument validation.  

In `build.gradle.kts`, we add 
`implementation("org.jetbrains.kotlinx:kotlinx-cli:0.3.5")` - 
[kotlinx](https://github.com/Kotlin/kotlinx-cli) argparsing library. We add the following to 
Main.kt:
```kotlin
val parser = ArgParser("example")
    val input by
            parser.option(ArgType.String, shortName = "i", description = "Input file").required()
    parser.parse(args)
    println("Input: $input")
```
The syntax is really close to python's `argparse` package.  
One interesting thing here is a "by" keyword, described in detail
[here](https://play.kotlinlang.org/byExample/07_Delegation/02_DelegatedProperties)
Compare to how we could've written that in, say, go:
```go
inputPtr := flag.String("input", "input.txt", "Input file")
```
Here, inputPtr is a pointer to a string. It is constructed in `flag.String` function, that can actually
read command line arguments and access the contents of the string through that pointer.  

This syntax is not ideal because   
a) we will need to dereference the pointer each time we use it and  
b) Kotlin doesn't have ponters )  

What we do here instead is using Delegate mechanic. We define a String whose get/set methods are
delegated to a `parser.option` function that reads cmd arguments and populates the string. 
Note that `input` is a `val`, i.e., constant, so we can't accidentally modify* it later. Neat!

This is how we can run it with some arguments: `./gradlew  run --args="-i input.txt"`

### Writing hadoop mapreduce code
Adding output option to main:
```kotlin
val output by parser.option(ArgType.String, fullName = "output", shortName = "o", description = "output directory").required()
```
And running a Hadoop job:
```kotlin
import kotlinx.cli.*

import org.apache.hadoop.conf.Configuration
import org.apache.hadoop.mapreduce.Job
import org.apache.hadoop.fs.Path
import org.apache.hadoop.io.IntWritable
import org.apache.hadoop.io.Text
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat
import org.apache.hadoop.mapreduce.lib.input.TextInputFormat
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat
import org.apache.hadoop.mapreduce.lib.output.TextOutputFormat

// MR job has to be instantiated in a class
// because Hadoop runtime expects a class entrypoint
object WordCount {
    fun run(input: String, output: String) : Boolean {
        val inputPath = Path(input)
        val outputPath = Path(output)
        val conf = Configuration(true)
        val job = Job.getInstance(conf)
        job.setJarByClass(WordCount::class.java)
        job.setMapperClass(TMapper::class.java)
        job.setReducerClass(TReducer::class.java)

        job.mapOutputKeyClass = Text::class.java
        job.mapOutputValueClass = IntWritable::class.java
        job.outputKeyClass = Text::class.java
        job.outputValueClass = IntWritable::class.java

        FileInputFormat.addInputPath(job, inputPath)
        job.setInputFormatClass(TextInputFormat::class.java)

        FileOutputFormat.setOutputPath(job, outputPath)
        job.setOutputFormatClass(TextOutputFormat::class.java)

        return job.waitForCompletion(true)
    }
}

fun main(args: Array<String>) {
    val parser = ArgParser("wordcount")
    val input by
    parser.option(ArgType.String, fullName = "input", shortName = "i", description = "Input file").required()
    val output by parser.option(ArgType.String, fullName = "output", shortName = "o", description = "output directory").required()
    parser.parse(args)

    val code = if (WordCount.run(input, output)) 0 else 1
    kotlin.system.exitProcess(code)
}
```
We haven't defined TMapper and TReducer class yet, we will do that later.  
Here is what is going on. Our program declares and launches a MapReduce job. "Launching" a job means talking to
some Hadoop master that will get job details - inputs, outputs, mapper and reducer class that actually do the work - 
making the master create and execute the job. Our own program will then ping the master periodically, making sure
that the job finishes successfully. This last part is optional - we could've just launched the job and forget about
it, trusting that Hadoop master will finish the job. Or we could launch a few jobs at once, and then wait for them
all to complete.  

`object WordCount {`: Hadoop library is written in Java, and its interface expect our job running code to be inside
a class. `object` kotlin keyword is perfect for this: it declaers a sort of "singleton" class that used to just 
wrap some functions, and does not require instantiation. Consider it a compatibility layer between kotlin and java 
that expect everything to be defined inside a class.  

```kotlin
val inputPath = Path(input)
val outputPath = Path(output)
```
Hadoop expects input and output paths to be passed as instances of `org.apache.hadoop.fs.Path`, so we do some 
converting.  

```kotlin
val conf = Configuration(true)
val job = Job.getInstance(conf)
```
This is how we initialize a new job.
```kotlin
job.setJarByClass(WordCount::class.java)
```
We need to pass a class that is used to initialize the job. That is why we defined it in a Wordcount object.
```kotlin
job.setMapperClass(TMapper::class.java)
job.setReducerClass(TReducer::class.java)
```
Set mapper/reducer for the job that will actually contain the processing logic. Implicitly, all Jobs in Hadoop are
mapreduce jobs. If you specify only a mapper, then it will be a Map job. You can't specify only a reducer, 
because mapper is used to split input into keys and values, and keys are used to "reduce by". Shuffle 
(or GroupByKey/CoGroupByKey for Apache Beam users) stage is implied before Reduce. Note that we also can't "bundle"
a few jobs together in a graph, like we could do in Beam. If we want multi-stage processing, we will need to define
and run a few Jobs, persisting intermediate outputs into HDFS (or other file system, like gcs or local). 
```kotlin
job.mapOutputKeyClass = Text::class.java
job.mapOutputValueClass = IntWritable::class.java
job.outputKeyClass = Text::class.java
job.outputValueClass = IntWritable::class.java
```
Every operation in Hadoop has to receive and output (key, value) pairs. Again, Hadoop is rather old-stylish in this
regard as compared to, say, Beam, where you could output single elements, pairs, tuples etc. Input and output key/value
types are templated, but they all must be of one of Hadoop's special "serializable" classes for persisting and 
passing around through network. Mapper expects 2 types (key/value) for input, reducer expects 2 types for output, 
so we set 4 different types. Reducer input is always the same as mapper output.
```kotlin
FileInputFormat.addInputPath(job, inputPath)
job.setInputFormatClass(TextInputFormat::class.java)

FileOutputFormat.setOutputPath(job, outputPath)
job.setOutputFormatClass(TextOutputFormat::class.java)
```
`FileInputFormat.add...` functions are used to set input and output paths for the job. They accept various filesystem
formats, including just local files "tmp/input.txt" as well as google file storage "gs://".  
`TextInputFormat` sets the file format to be used. It expects a text file, splits it into lines with `\n` separator,
and feeds line number as key and the actual line as a value to the mapper.
`return job.waitForCompletion(true)` actually starts the job and tells the program to wait for it to complete before
returning.  

Also note that throughout this function, we were using Java functions directly from our Kotlin code.
### Mapper class
```kotlin
import org.apache.hadoop.io.IntWritable
import org.apache.hadoop.io.LongWritable
import org.apache.hadoop.io.Text
import org.apache.hadoop.mapreduce.Mapper
import java.util.regex.Pattern

class TMapper : Mapper<LongWritable, Text, Text, IntWritable>() {
    public override fun map(key: LongWritable, value: Text, context: Context) {
        val words = value.toString().split(Pattern.compile("\\W+"), limit = 0)
        words.map(String::lowercase)
            .forEach { context.write(Text(it), IntWritable(1)) }
    }
}
```
To define a Mapper, we define a class that inherits from a Mapper class, and override a `map` function.
Input types for the map function have to be the same as we declared in the job.
`map` function receives just one record. We can override additional functions `setup` and `cleanup` 
to execute some code before and after processing the records. Alternatively, we can override the `run`
function to get a complete control over what happens in a mapper - for example, iterate over all input
records ourselves. Note that even in this case we are not guaranteed to receive ALL records for the same
job - because they could've gone to a different mapper.  

To write output, we use `context.write` and pass it (key, value) pair, converted to Hadoop serializable
types. Note that we can call it any number of times - 0, 1 or more - for one input value.  

Actual mapper logic is: receive (lineNumber, lineText) from `TextInputFormat` file parser; split lineText
by into words (treating anything space-separated as a word); outputting (word, 1) pairs for every word.
### Reducer class
```kotlin
import org.apache.hadoop.io.IntWritable
import org.apache.hadoop.io.Text
import org.apache.hadoop.mapreduce.Reducer

class TReducer : Reducer<Text, IntWritable, Text, IntWritable>() {
    override fun reduce(key: Text, values: Iterable<IntWritable>, context: Context) {
        val sum = values.sumOf(IntWritable::get)
        context.write(key, IntWritable(sum))
    }
}
```
Almost nothing new is happening here. We inherit from Reducer class, override `reduce` function. 
It receives one key and ALL values for that key (which is the whole point of Reduce!). 
### Running locally
Running `gradlew jar` will produce a jar for us under `build/lib/`. We need to then feed it to the
actual hadoop executer.  

To install hadoop, we just go to the [official website](https://hadoop.apache.org/releases.html), download
the `.tar.gz` file, extract it and put somewhere we put all our other programs. In my case, that is
`~/bin`. We then need to point our PATH variables to `~/bin/hadoop-3.3.5/bin` (your version can be 
different).   

And now actually run it:
```kotlin
# generate some input data
echo "word1 word2" > input.txt
# make sure output directory doesnt exist
rm -r output
# run a hadoop job!
hadoop jar build/libs/wordcount-1.0-SNAPSHOT.jar -i input.txt -o output
```
The job runs, and in `output/part-r-00000` we can see the following:
```kotlin
word1	1
word2	1
```
Congradulations, it works!  

Note that to be actually able to run this on Mac, I also had to add `exclude("META-INF/*")` to
`tasks.jar` section in `build.gradle.kts` because I was getting some META-INF related errors.  

Also note that when we run `hadoop jar build/libs/wordcount-1.0-SNAPSHOT.jar -i input.txt -o output`,
there are 2 entities running on our machine: the Hadoop executor that takes our program as input,
sends it to a Hadoop cluster, and waits for execution; and a hadoop cluster itself that was just created
locally. This mode is useful for debugging only, normally we will be communicating with an actual remote
cluster with potentially thousands of workers doing the processing.
### gcs connector for local debugging
Next step is to be able to run jobs with gcs files as inputs/outputs. On Dataproc, that will work
out of the box. For local development, some additional connectors need to be installed.  

We will only consider the case of running a local job on a gcp-vm - it already has all the required 
credentials to access gcs.   

Here is the [connector](https://cloud.google.com/dataproc/docs/concepts/connectors/cloud-storage). 
To install it, simply download the connector for a specific hadoop version that you have; this will 
download the `.jar` file. Then put this jar file into what is known as HADOOP_COMMON_LIB_JARS_DIR.
It should be located at `$HADOOP_BASE/share/hadoop/common/lib/`, where $HADOOP_BASE is where you
put your hadoop installation. After that, this command should work:  
`hadoop fs -ls gs://<bucketname>`  

Now we can run our local hadoop job with gcs as input and output:
```bash
# uploading input file to gcs
gsutil cp input.txt gs://<bucketname>/input.txt
# make sure that outpout directory doesn't exist!
hadoop jar build/libs/wordcount-1.0-SNAPSHOT.jar -i gs://mgaiduk/input.txt -o gs://mgaiduk/output123 
```
### Running on dataproc
Dataproc is a Google technology to launch Hadoop clusters in GCP with just one easy step.   
To create a cluster:
```bash
export PROJECT=my_project # id of your actual gcp project
export BUCKET_NAME=my_bucket # Your actual existing gcs bucket name. Will be used for running jobs
export CLUSTER=my-dataproc-cluster # any name you  want
export REGION=us-central1
# WARNING! Billable event
gcloud dataproc clusters create ${CLUSTER} \
	    --project=${PROJECT} \
	    --region=${REGION} \
        --labels <...>\ # labels can be used for additional cost attribution
	    --single-node
```
Creating a cluster takes about 2 minutes, so it shouldn't be a big problem to just create one whenever
you need some data processed.
`--single-node` mode option will create a cluster with master and all workers on one node. Without it,
a master + 8 workers configuration will be created. To create a custom configuration:
```bash
# export current cluster configuration
gcloud dataproc clusters export $CLUSTER --region $REGION --destination cluster.yaml 
# create a new cluster after modifying the configuration
gcloud dataproc clusters import $CLUSTER --source cluster.yaml --region=$REGION
```
Since you are billed for the cluster even if no jobs are running, don't forget to delete the cluster
afterwards:
```bash
gcloud dataproc clusters delete $CLUSTER --region=$REGION
```
To submit a job:
```bash
gcloud dataproc jobs submit hadoop --cluster=$CLUSTER --region $REGION --jar=build/libs/wordcount-1.0-SNAPSHOT.jar --\
 -i gs://mgaiduk/input.txt -o gs://mgaiduk/output1234
```
One thing to note is that Google Dataproc uses a specific hadoop version. Current available and
default versions can be seen [here](https://cloud.google.com/dataproc/docs/concepts/versioning/dataproc-release-2.0).
We can see the dataproc image with which our cluster was created in the `cluster.yaml` file.  
In our case, the cluster version was `https://www.googleapis.com/compute/v1/projects/cloud-dataproc/global/images/dataproc-2-0-deb10-20230327-165100-rc01`,
main thing here being `20230327`. This image corresponds to `3.2.3` hadoop version and `8` Java version.
We have to make necessary updates in our `build.gradle.kts`.
---
We now know how to write some basic Hadoop code and to launch it both locally and on Google Dataproc.
That one actually gives us a lot of possibilities - launching a huge data processing job with our
custom code that could exploit potentially thousands of workers and process petabytes of data - provided
we have enough money to pay for that )  

In the next chapter, we will create a more advanced Hadoop program to do some useful feature simulations for
a RecSys/ML project.