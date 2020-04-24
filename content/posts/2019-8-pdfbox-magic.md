+++
title = "PDFBox searching"
description = "Search through PDFs with PDFBox and leverage Tika for mime-type verification"
date = "2019-08-22"
tags = ["groovy"]
+++

Sometimes you need to search through bank statements looking for that [Sports Basement](https://shop.sportsbasement.com)
receipt you need to replace some expensive hiking / climbing gear which is untracked
by your own or their systems... That's what took me down a path to creating this little gem
for doing so. The version I created after this leveraged a [fuzzy search](https://github.com/xdrop/fuzzywuzzy)
to examine document payloads instead of string literals.

```java
@Grapes([
  @Grab(group='org.apache.pdfbox', module='pdfbox', version='2.0.16'),
  @Grab(group='org.apache.tika', module='tika-core', version='1.22')
])

import org.apache.pdfbox.pdmodel.PDDocument
import org.apache.pdfbox.text.PDFTextStripper
import org.apache.tika.Tika
import java.util.concurrent.ThreadPoolExecutor
import java.util.concurrent.TimeUnit
import java.util.concurrent.LinkedBlockingQueue
import java.util.concurrent.ThreadFactory
import java.lang.ThreadLocal

import static groovy.io.FileType.FILES

def printerr = System.err.&println

def cli = new CliBuilder(header: 'PDF Text Extractor', usage:'./pdfTextExtractor <directoryToScan> <phrase>', width: 100)

def cliOptions = cli.parse(args)

if (cliOptions.help || cliOptions.arguments().size() != 2) {
  cli.usage()
  System.exit(0)
}

class TextExtractor {
  static String requiredMimeType = 'application/pdf'
  def pdfStripper = new PDFTextStripper()
  def tika = new Tika()

  void processFile(File file) {

    try {

      if (tika.detect(file) == requiredMimeType) {
        def document = PDDocument.load(file)
        def text = pdfStripper.getText(document)

        text.eachLine { line ->

         if (line.contains(cliOptions.arguments().last())) {
            println line
          }
        }

        document.close()
      }
    } catch (Exception e) {
      printerr "Error processing file ${file.path}"
    }
  }
}

class ReusableThread extends Thread {
  static def extractor = ThreadLocal.withInitial({
    new TextExtractor()
  })
  ReusableThread(Runnable runnable) {
    super(runnable)
  }
  void run() {
    super.run()
  }
}

class ReusableThreadFactory implements ThreadFactory {
  Thread newThread(Runnable runnable) {
    new ReusableThread(runnable)
  }
}

def executor = new ThreadPoolExecutor(5, 50, 5, TimeUnit.SECONDS, new LinkedBlockingQueue(50), new ReusableThreadFactory(), new ThreadPoolExecutor.CallerRunsPolicy())

new File(cliOptions.arguments().first()).eachFileRecurse(FILES) { file ->

  executor.submit {

    def extractor = ReusableThread.extractor.get()
    extractor.processFile(file)
  }
}

executor.shutdown()
```

`groovy DocumentScanner.groovy ~/Documents/receipts "sports basement" > results.txt 2>/dev/null & tail -100f results.txt`
