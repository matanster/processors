# What is it

The `edu.arizona.sista.processor` package aims to be a one-stop place for natural language (NL) processors.
We currently provide a Scala API for 
[Stanford's CoreNLP](http://nlp.stanford.edu/software/corenlp.shtml) 
but plan to add more in the future, including tools developed in house.

This software requires Java 1.6, Scala 2.9 or higher, and CoreNLP 1.3.4 or higher. 

This code is licensed under GPL v2 or higher.

(c) Mihai Surdeanu, 2013 - 

# Changes
+ **1.4** - Code cleanup. Added some minor new functionality such as finding base NPs in the Trees class.
+ **1.3** - Reverted back to the `1.x` version numbers, since we will add other software here not just CoreNLP. Added correct mvn dependencies for the CoreNLP jars. Removed the `install*.sh` scripts, which are no longer needed.
+ **3.2.0** - Updated to Scala 2.10.1 and CoreNLP 3.2.0. Changed versioning system to be identical to CoreNLP's, so it's clear which CoreNLP version is used.
+ **1.0** - Initial release

# Maven
This software is available on maven as well. Add this dependency to your `pom.xml` to use it:

    <dependency>
       <groupId>edu.arizona.sista</groupId>
       <artifactId>processors</artifactId>
       <version>1.4</version>
    </dependency>

# Why you should use this code
+ **Simple API** - the APIs provided are, at least in my opinion, simpler than those provided for the original code. For example, when using CoreNLP you won't have to deal with hash maps that take class objects as keys. Instead, we use mostly arrays of integers or strings, which are self explanatory.
+ **Memory efficient** - arrays are more memory efficient than hash maps. Furthermore, we used our own implementation to intern strings (i.e., avoiding to store duplicated strings multiple times). Due to these changes, I measured up to 99% decrease in memory for the annotations corresponding to a typical natural language text, when compared to the original CoreNLP code. (Note: this reduction takes effect only _after_ CoreNLP finishes its work.)
+ **Faster access** - again, because we use arrays instead of hash maps, access to the NL annotations (once constructed) is considerably faster than in the original Stanford code.
+ **Tool-independent API** - we plan to support multiple NL tools in the future. However, if you use this code, you will have to change your code only minimally (i.e., the constructor for the `Processor` object).

# How to compile 

This is a standard Maven project, so use the `mvn package` command to build the jar file, which will be stored in the `target/` directory.
Add the generated jar file to your $CLASSPATH, along with the jar files for CoreNLP.

# How to use it

## Common scenarios

### Annotating entire documents

    // create the processor
    val proc:Processor = new CoreNLPProcessor()

    // the actual work is done here
    val doc = proc.annotate("John Smith went to China. He visited Beijing, on January 10th, 2013.")

    // you are basically done. the rest of this code simply prints out the annotations

    // let's print the sentence-level annotations
    var sentenceCount = 0
    for (sentence <- doc.sentences) {
      println("Sentence #" + sentenceCount + ":")
      println("Tokens: " + sentence.words.mkString(" "))
      println("Start character offsets: " + sentence.startOffsets.mkString(" "))
      println("End character offsets: " + sentence.endOffsets.mkString(" "))

      // these annotations are optional, so they are stored using Option objects, hence the foreach statement
      sentence.lemmas.foreach(lemmas => println("Lemmas: " + lemmas.mkString(" ")))
      sentence.tags.foreach(tags => println("POS tags: " + tags.mkString(" ")))
      sentence.entities.foreach(entities => println("Named entities: " + entities.mkString(" ")))
      sentence.norms.foreach(norms => println("Normalized entities: " + norms.mkString(" ")))
      sentence.dependencies.foreach(dependencies => {
        println("Syntactic dependencies:")
        val iterator = new DirectedGraphEdgeIterator[String](dependencies)
        while(iterator.hasNext) {
          val dep = iterator.next
          // note that we use offsets starting at 0 (unlike CoreNLP, which uses offsets starting at 1)
          println(" head:" + dep._1 + " modifier:" + dep._2 + " label:" + dep._3)
        }
      })
      sentence.syntacticTree.foreach(tree => {
        println("Constituent tree: " + tree)
        // see the edu.arizona.sista.utils.Tree class for more information
        // on syntactic trees, including access to head phrases/words
      })

      sentenceCount += 1
      println("\n")
    }

    // let's print the coreference chains
    doc.coreferenceChains.foreach(chains => {
      for (chain <- chains.getChains) {
        println("Found one coreference chain containing the following mentions:")
        for (mention <- chain) {
          // note that all these offsets start at 0 too
          println("\tsentenceIndex:" + mention.sentenceIndex +
            " headIndex:" + mention.headIndex +
            " startTokenOffset:" + mention.startOffset +
            " endTokenOffset:" + mention.endOffset +
            " text: " + doc.sentences(mention.sentenceIndex).words.slice(mention.startOffset, mention.endOffset).mkString("[", " ", "]"))
        }
      }
    })
    
The above code generates the following output:

    Sentence #0:
    Tokens: John Smith went to China .
    Start character offsets: 0 5 11 16 19 24
    End character offsets: 4 10 15 18 24 25
    Lemmas: John Smith go to China .
    POS tags: NNP NNP VBD TO NNP .
    Named entities: PERSON PERSON O O LOCATION O
    Normalized entities: O O O O O O
    Syntactic dependencies:
      head:1 modifier:0 label:nn
      head:2 modifier:1 label:nsubj
      head:2 modifier:4 label:prep_to
    Constituent tree: (ROOT (S (NP (NNP John) (NNP Smith)) (VP (VBD went) (PP (TO to) (NP (NNP China)))) (. .)))


    Sentence #1:
    Tokens: He visited Beijing , on January 10th , 2013 .
    Start character offsets: 26 29 37 44 46 49 57 61 63 67
    End character offsets: 28 36 44 45 48 56 61 62 67 68
    Lemmas: he visit Beijing , on January 10th , 2013 .
    POS tags: PRP VBD NNP , IN NNP JJ , CD .
    Named entities: O O LOCATION O O DATE DATE DATE DATE O
    Normalized entities: O O O O O 2013-01-10 2013-01-10 2013-01-10 2013-01-10 O
    Syntactic dependencies:
      head:1 modifier:0 label:nsubj
      head:1 modifier:2 label:dobj
      head:1 modifier:8 label:tmod
      head:2 modifier:5 label:prep_on
      head:5 modifier:6 label:amod
    Constituent tree: (ROOT (S (NP (PRP He)) (VP (VBD visited) (NP (NP (NNP Beijing)) (, ,) (PP (IN on) (NP (NNP January) (JJ 10th))) (, ,)) (NP-TMP (CD 2013))) (. .)))


    Found one coreference chain containing the following mentions:
      sentenceIndex:1 headIndex:2 startTokenOffset:2 endTokenOffset:3 text: [Beijing]
    Found one coreference chain containing the following mentions:
      sentenceIndex:1 headIndex:0 startTokenOffset:0 endTokenOffset:1 text: [He]
      sentenceIndex:0 headIndex:1 startTokenOffset:0 endTokenOffset:2 text: [John Smith]
    Found one coreference chain containing the following mentions:
      sentenceIndex:1 headIndex:5 startTokenOffset:5 endTokenOffset:9 text: [January 10th , 2013]
    Found one coreference chain containing the following mentions:
      sentenceIndex:0 headIndex:4 startTokenOffset:4 endTokenOffset:5 text: [China]

For more details about the annotation data structures, please see the `edu/arizona/sista/processor/Document.scala` file.

### Annotating documents already split into sentences

    val doc = proc.annotateFromSentences(List("John Smith went to China.", "He visited Beijing."))
    // everything else stays the same

### Annotating documents already split into sentences and tokenized

    val doc = annotateFromTokens(List(
      List("John", "Smith", "went", "to", "China", "."),
      List("There", ",", "he", "visited", "Beijing", ".")))
    // everything else stays the same

## Using individual annotators

You can of course use only some of the annotators provided by CoreNLP by calling them individually. To illustrate, 
the `Processor.annotate()` method is implemented as follows:

    def annotate(doc:Document): Document = {
      tagPartsOfSpeech(doc)
      lemmatize(doc)
      recognizeNamedEntities(doc)
      parse(doc)
      chunking(doc)
      labelSemanticRoles(doc)
      resolveCoreference(doc)
      doc.clear()
      doc
    }

(Note that CoreNLP currently does not support chunking and semantic role labeling.)
You can use just a few of these annotators. For example, if you need just POS tags, lemmas and named entities, you could use 
the following code:

    val doc = proc.mkDocument("John Smith went to China. He visited Beijing, on January 10th, 2013.")
    proc.tagPartsOfSpeech(doc)
    proc.lemmatize(doc)
    proc.recognizeNamedEntities(doc)
    doc.clear()
    
Note that the last method called (`doc.clear()`) clears the internal structures created by the actual CoreNLP annotators. 
This saves a lot of memory, so, although it is not strictly necessary, I recommend you call it.

## Serialization

This package also offers serialization code for the generated annotations. The two scenarios currently supported are:

### Serialization to/from streams

    // saving to a PrintWriter, pw
    val someAnnotation = proc.annotate(someText)
    val serializer = new DocumentSerializer
    serializer.save(someAnnotation, pw)

    // loading from an InputStream, is
    val someAnnotation = serializer.load(is)

### Serialization to/from Strings
    
    // saving to a String object, savedString
    val someAnnotation = proc.annotate(someText)
    val serializer = new DocumentSerializer
    val savedString = serializer.save(someAnnotation)
    
    // loading from a String object, fromString
    val someAnnotation = serializer.load(fromString)

Note that space required for these serialized annotations is considerably smaller (8 to 10 times) than the corresponding 
serialized Java objects. This is because we store only the information required to recreate these annotations (e.g., words, lemmas, etc.)
without storing any of the Java/Scala objects and classes. 

## Cleaning up the interned strings

Classes that implement the `Processor` trait intern String objects to avoid allocating memory repeatedly for the same string.
This is implemented by maintaining an internal dictionary of strings previously seen. This dictionary is unlikely to 
use a lot of memory due to the Zipfian distribution of language. But, if memory usage is a big concern, it can be cleaned by 
calling:

    Processor.in.clear()

I recommend you do this only _after_ you annotated all the documents you plan to keep in memory.

Although I see no good reason for doing this, you can disable the interning of strings completely by setting the `internStrings = false` in the 
CoreNLProcessor constructor, as such:

    val processor = new CoreNLPProcessor(internStrings = false)



