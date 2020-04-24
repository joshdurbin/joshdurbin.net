+++
title = "Leveraging Groovy, Stanford NLP, and SQLite to performance sentiment analysis on iMessages"
date = "2019-10-15"
tags = ["sqlite", "groovy", "nlp"]
+++

Over the weekend I crafted this little gem to run the [Stanford Core NLP](https://stanfordnlp.github.io/CoreNLP/) libraries against my messages
stored in iMessage on my Mac.

The script is pretty straight forward...

1. it obtains a hold on the [SQLite](https://sqlite.org/index.html) DB used for Message storage
2. queries the `messages` tables for its `text` value referring to any incoming or outgoing message within the DB for conversations
3. runs [sentiment analysis](https://en.wikipedia.org/wiki/Sentiment_analysis) against the text
4. pretty prints the overall sentiment using groovy [metaprogramming](http://groovy-lang.org/metaprogramming.html)

```
@Grapes([
    @Grab(group='org.xerial', module='sqlite-jdbc', version='3.27.2.1'),
    @Grab(group='edu.stanford.nlp', module='stanford-corenlp', version='3.9.2'),
    @Grab(group='edu.stanford.nlp', module='stanford-corenlp', version='3.9.2', classifier='models')
])
@GrabConfig(systemClassLoader=true)

import java.math.BigDecimal
import java.math.MathContext
import java.util.Properties
import java.math.RoundingMode

import groovy.sql.Sql

import edu.stanford.nlp.pipeline.StanfordCoreNLP
import edu.stanford.nlp.ling.CoreAnnotations
import edu.stanford.nlp.neural.rnn.RNNCoreAnnotations
import edu.stanford.nlp.pipeline.Annotation
import edu.stanford.nlp.pipeline.StanfordCoreNLP
import edu.stanford.nlp.sentiment.SentimentCoreAnnotations
import edu.stanford.nlp.trees.Tree
import edu.stanford.nlp.util.CoreMap

import org.ejml.simple.SimpleMatrix

def props = new Properties()
props.setProperty("annotators", "tokenize, ssplit, parse, sentiment")
def pipeline = new StanfordCoreNLP(props)

def sql = Sql.newInstance('jdbc:sqlite:Library/Messages/chat.db', 'org.sqlite.JDBC')

def mc = new MathContext(2, RoundingMode.HALF_DOWN)

SimpleMatrix.metaClass.getPretty { def index ->
  new BigDecimal(delegate.get(index), mc).movePointRight(2) as String
}

sql.eachRow('select text from message') { message ->
  def text = message.text
  if (text) {  

    def annotation = pipeline.process(text)

    for (def sentence : annotation.get(CoreAnnotations.SentencesAnnotation)) {

      def tree = sentence.get(SentimentCoreAnnotations.SentimentAnnotatedTree)
      def sm = RNNCoreAnnotations.getPredictions(tree)
      def sentimentType = sentence.get(SentimentCoreAnnotations.SentimentClass)

      println "----> '${sentence}' is overall ${sentimentType}"
      println "w/ %${sm.getPretty(0)} very negative, %${sm.getPretty(1)} negative, %${sm.getPretty(2)} neutral, %${sm.getPretty(3)} positive %${sm.getPretty(4)} very positive"
      println ""
    }
  }
}
```

...producing results that look like:

```
➜  ~ groovy nlpMessagesAnalysis.groovy
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
----> 'Uh, ok, I’m running way later than that...' is overall Negative
w/ %9.9 very negative, %54 negative, %26 neutral, %7.2 positive %2.4 very positive

----> 'Where will I find you?' is overall Neutral
w/ %2.6 very negative, %24 negative, %54 neutral, %18 positive %1.5 very positive

----> 'Shit I didn’t this until now' is overall Negative
w/ %7.5 very negative, %42 negative, %33 neutral, %14 positive %3.5 very positive

----> 'Are you here?' is overall Neutral
w/ %0.69 very negative, %14 negative, %82 neutral, %3.3 positive %0.16 very positive

----> 'No, I’m still on my way, running super late' is overall Negative
w/ %16 very negative, %44 negative, %34 neutral, %3.8 positive %1.8 very positive

----> 'There were no lyfts in bayview...' is overall Negative
w/ %5.5 very negative, %48 negative, %40 neutral, %6.5 positive %0.84 very positive

----> 'I’m hoping they’ll start the show late - I don’t want to miss Jason and I know you said he’s performing in the first hour!!' is overall Negative
w/ %23 very negative, %58 negative, %14 neutral, %2.8 positive %2.1 very positive

----> '😭' is overall Neutral
w/ %2.9 very negative, %14 negative, %63 neutral, %15 positive %4.3 very positive

```
