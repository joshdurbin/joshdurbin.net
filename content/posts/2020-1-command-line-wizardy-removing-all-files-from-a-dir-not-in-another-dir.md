+++
title = "Using Groovy to achieve something that I should be able to do in a single unix, pipped set of commands"
date = "2020-01-1"
tags = ["groovy", "work-arounds"]
+++

The task is simple; I want to delete all cards off my DSLRs camera SD card that aren't in Lightroom and thus NOT WORTHY of existing anymore.

I turned to Groovy to make this happen, but should have been able to do something with standard Unix command line tools.

```groovy
import groovy.io.FileType

def importedFiles = []

new File('/Users/jdurbin/Pictures/Lightroom Library.lrlibrary').eachFileRecurse (FileType.FILES) { file ->
  if (file.path.contains('NEF')) {
    importedFiles << file.path
  }
}

new File('/Volumes/NIKON D3500').eachFileRecurse (FileType.FILES) { file ->
  if (file.path.contains('NEF')) {
    if (!importedFiles.contains(file.path)) {
      println "Deleting ${file.path}..."
      file.delete()
    }
  }
}
```
