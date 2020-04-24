+++
title = "Groovy Snippets: Content detection w/ tika"
date = "2017-05-15"
tags = ["groovy"]
+++

A few months back I had to process a few million, a few terrabytes of assets that
were missing content extensions. I needed to filter the assets so that
I could stream assets that were images to 3rd party provider's API for further analysis. Rough
estimates for the number of images in the few million assets came out to no more
than about an 1/8. Fortunately I had access to the data on a particular machine and used
Tika to quickly zip through the assets and identify those that met my required
content types. The script is super simple and identifies PNG and JPEG assets:

```
#!/usr/bin/env groovy

@Grapes([
    @Grab(group='org.apache.tika', module='tika-core', version='1.14')
])

import static groovy.io.FileType.FILES

import org.apache.tika.Tika

def tika = new Tika()
def acceptableImageTypes = ['image/jpeg', 'image/png']

new File('/vol/data/assets').eachFileRecurse(FILES) { file ->

  def detectedType = tika.detect(file)

  if (acceptableImageTypes.contains(detectedType)) {

    println "file '${file.path}' is content type '${detectedType}'."
  }
}
```