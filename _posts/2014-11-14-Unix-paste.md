---
layout: post
category : linux
tagline: "for FASTQ and FASTA files"
tags : [linux, command line, bash, tutorial, paste]
---
{% include JB/setup %}

This week has seen the Australian Centre for Ancient DNA ([ACAD](https://www.adelaide.edu.au/acad/)) run their "[Advanced Bioinformatics Workshop for Early Career Researchers](https://www.adelaide.edu.au/acad/events/bioinformatics14/)".
I was lucky enough to be invited out to dinner with the speakers the other night. As is usual with such events, there was some talk of
work and what programming languages people knew and currently use. To paraphrase my response:

"I first started using Perl but find myself using bash more and more often because the power and speed of the Unix command line is awsome"

As I went on, I mentioned my use of the Unix `paste` command. Both [Dan Huson](http://ab.inf.uni-tuebingen.de/people/huson/) and
[Stephan Schiffels](https://twitter.com/stschiff) hadn't really fully understood the awsomeness of `paste` and thought it was pretty cool and suggested I blogged about it. So
here I am writing my first blog post in my new blog "Gluten for Punishment: A Bioinformaticians Blog".

## Overview

Before going into the specifics of `paste` I want to say a few words about the Unix philosophy, how many file formats encountered in
bioinformatics do not lend themselves well to this philosophy and finally how paste can be used to improve the situation with FASTA and
FASTQ files.

### Unix philosophy

The [Unix philosophy](http://en.wikipedia.org/wiki/Unix_philosophy#Doug_McIlroy_on_Unix_programming) states:
"*Write programs that do one thing and do it well. Write programs to work together. Write programs to handle text streams, because that is a universal interface.*"
In a nut-shell, write small programs that are capable of reading from `STDIN` and writing to `STDOUT`. 

### Streams

Unix uses the end-of-line (EOL) character as the default record separator. That is, lines of text are read in one at a
time on `STDIN` and written to `STDOUT` one at a time. Most Unix tools abide by the above Unix philosophy and process `STDIN` and `STDOUT` with the "one line = one
record" concept in mind and are blazingly fast at processing such data. However, if in your file format a record occupies
more than one line e.g. **FASTQ** and **FASTA** you often need to have some programming logic to deal with that fact and this
costs operations and time. This can result in quite a large processing (time) overhead and also means you miss out on
using cool, fast tools that expect a record to occupy a single line.

This is where [`paste`](http://linux.die.net/man/1/paste) comes in.

### What is paste?

`paste` is a lesser known command found on most (all) Unix/Linux computers. The documentation
simply states:

"*merge lines of files*"

The description is somewhat more helpful but still a bit cryptic:

"*Write lines consisting of the sequentially corresponding lines from each FILE, separated by TABs, to standard output.
With no FILE, or when FILE is -, read standard input.*"

Essentially, `paste` displays the corresponding lines of multiple files side-by-side i.e. creates tab-delimited output where column 1
are the lines from file 1, column 2 are the lines from file 2 etc.

## Examples

Rather than describe what `paste` does, lets take a look at some quick examples.

### FASTQ Processing

We assume that a FASTQ record occupies 4 lines. This is a reasonable assumption and I have yet to see a FASTQ file which breaks it.

{% highlight bash %}
# Let's see what our file contains
$ cat my.fastq
@my_seq_id_1
ACTGGGCC
+
AAAAAAAA
@my_seq_id_2
ACGGTCGT
+
AAAAAAAA

# Use paste on our file
$ paste - - - - < my.fastq
@my_seq_id_1	ACTGGGCC	+	AAAAAAAA
@my_seq_id_2	ACGGTCGT	+	AAAAAAAA
{% endhighlight %}

You'll notice that the output is tab-delimited where column 1 contains the 1st line of the record (the sequence ID line),
column 2 contains the 2nd line of the record (the sequence itself), column 3 contains the 3rd line of the record (the '+') and
column 4 contains the 4th line of the record (the quality string).

After using `paste` each sequence record now occupies a single line, and we can now bring the power of native Unix
tools to the FASTQ sequences. In particularly useful thing to do is to apply some sort of [filter](http://www.linfo.org/filters.html)
to the reads, since this would have previously required additional logic to deal with the 4 lines of a FASTQ
record. So, lets filter for reads containing the sequence `ACTG`:

{% highlight bash %}
# Lets do the filter
paste - - - - < my.fastq | fgrep 'ACTG'
@my_seq_id_1	ACTGGGCC	+	AAAAAAAA

# Lets do this again, but reformat back to the traditional FASTQ format of 4 lines per record
paste - - - - < my.fastq | fgrep 'ACTG'
@my_seq_id_1
ACTGGGCC
+
AAAAAAAA
{% endhighlight %}

So, what's happening here? Well, many Unix tools can interpret `-` as shorthand for `STDIN` or `STDOUT`. Contrary
to what some people think `-` is not a Bash operator. See The Linux Documentation Projects' website
[Advanced Bash-Scripting Guide](http://www.tldp.org/LDP/abs/html/special-chars.html#DASHREF2) for more information.

### Paired-end FASTQ

Paired-end FASTQ data comes in one of two ways:

  1. The reads from the pairs come in separate files
  2. The reads from the pairs come in the same file but are interleaved

Using `paste` and a few other native Unix commands we can write some fast converters for interleaving and deinterleaving
paired-end data. The advantage of this is we can apply filters which easily maintains read pairs without any
programming logic overhead.

{% highlight bash %}
# Let's see what our file contains
$ cat second.fastq
@my_seq_id_1/1
ACTGGGCC
+
AAAAAAAA
@my_seq_id_2/1
ACGGTCGT
+
AAAAAAAA

$ cat first.fastq
@my_seq_id_1/2
GTCATGAC
+
AAAAAAAA
@my_seq_id_2/2
TGCCCGAT
+
AAAAAAAA

# Interleave two FASTQ files
# The two paste commands result in a tab-delimited output containing 8 columns,
# 1-4 being for first read in the pair and 5-8 being the second read in the pair.
# We use awk to simply output the columns in the correct order for the interleaved output
$ paste forward.fastq reverse.fastq \
  | paste - - - - \
  | awk -v OFS="\n" -v FS="\t" '{print($1,$3,$5,$7,$2,$4,$6,$8)}'
@my_seq_id_1/1
ACTGGGCC
+
AAAAAAAA
@my_seq_id_1/2
GTCATGAC
+
AAAAAAAA
@my_seq_id_2/1
ACGGTCGT
+
AAAAAAAA
@my_seq_id_2/2
TGCCCGAT
+
AAAAAAAA

# Deinterleave a FASTQ file
# We're now also using tee and process substitution to run two different
# pipelines on the same stream of data coming into tee
paste - - - - - - - -  < interleaved.fastq \
  | tee >(cut -f 1-4 | tr "\t" "\n" > first.fastq) \
  | cut -f 5-8 \
  | tr "\t" "\n" \
  > second.fastq

{% endhighlight %}

Lets use a filter on our paired-end data which ensures we maintain read pairings.

{% highlight bash %}
# Lets filter the pairs so we only keep pairs where at least one of the reads in a pair must contain TGCC
$ paste forward.fastq reverse.fastq \
  | paste - - - - \
  | fgrep 'TGCC' \
  | awk -v OFS="\n" -v FS="\t" '{print($1,$3,$5,$7,$2,$4,$6,$8)}' \
  > interleaved_filtered_pairs.fastq
{% endhighlight %}

I have two gists which implement the interleaving and deinterleaving code above:

{% gist 4544979 %}

The following deinterleaving script optionally GZIP compresses the deinterleaved output FASTQ files.

{% gist 3521724 %}

### FASTA Processing

The sequences of FASTA files can wrap onto multiple lines, so do not easily lend themselves to the approach I
introduced above. However, a simple conversion, using one of my gists, can remedy the situation so that
FASTA records occupy 2 lines:

{% gist ba6264aa6ef70db4e743 %}

### Usage with FASTA Files

Once, we've put our FASTA sequences chunks onto a single line, our FASTA records now occupy 2 lines each and we can use
`paste` similarly to what we did before:

{% highlight bash %}
# Lets see what our file contains
cat my.fasta
>my_seq_id_1
ACT
G
GGCC
>my_seq_id_2
A
CGG
TCGT

# Now lets get all those sequences to occupy a single line
mlFASTA2slFASTA.sh < my.fasta
>my_seq_id_1
ACTGGGCC
>my_seq_id_2
ACGGTCGT

# Again, lets filter the sequences for those containing ACTG
mlFASTA2slFASTA.sh < my.fasta | paste - - | fgrep 'ACTG' | '\t' '\n'
>my_seq_id_1
ACTGGGCC
{% endhighlight %}

Assuming you are on a multi-core machine, each process in the above pipeline is run in parallel. Because each of the
processes is doing a small simple job without complicated programming logic, it's also lightning fast!

## Summary

Here is a summary of the main points you should remember:

  1. `paste` brings the power of the Unix shell to FASTA and FASTQ files by putting 1 record on each line
  2. Each process in a pipeline runs on a different CPU, so things are being processed in parallel (assuming
  you have sufficient cores)
  3. You can scrap much of the programming logic required to process FASTA and FASTQ files when they occupy more than
  one line per record.
  
## Other Tools to Take a Look at

Once you get a hang of the above, and if you have lots of cores on your machine, you should take a look at the following:

  1. [GNU Parallel](http://www.gnu.org/software/parallel/) for carving up the stream of FASTA/FASTQ records into chunks for
parallel processing.
  2. [`tre-agrep`](https://github.com/laurikari/tre/) for doing "approximate grep" (fuzzy matching) with `TRE` extensions. Essentially,
  you can specify the number of insertions, deletions, substitutions, the total numbers of errors as well as
  the cost equation.
  
Using a combination of the above it's easy to write a parallelised pipline for filtering sequences using
fuzzy matching **- the topic of a future blog post**?
  