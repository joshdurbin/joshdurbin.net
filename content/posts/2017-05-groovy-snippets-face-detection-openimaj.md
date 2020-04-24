+++
title = "Groovy Snippets: Face detection with OpenIMAJ"
date = "2017-05-17"
tags = ["groovy"]
+++

Building on the previous [post]({{< ref "/posts/2017-05-groovy-snippets-file-content-detection.md" >}}), this snippet leverages concurrent
execution and [OpenIMAJ](http://openimaj.org) for facial detection and analysis.

OpenIMAJ implements [types](http://openimaj.org/tutorial/pt07.html) a variety of facial
detection algorithms in its library. For this exercise, we use the:

- HaarCascadeDetector
- Frontal Keypoint Enhanced, which wraps the `HaarCascadeDetector`

The results of the script were used to narrow the possible matches within the dataset prior
to execution, leveraging cloud infrastructures.

```
#!/usr/bin/env groovy

@GrabResolver(name='OpenIMAJ Maven Repo', root='http://maven.openimaj.org')
@GrabResolver(name='tick', root='https://mvnrepository.com/')
@GrabConfig(systemClassLoader= true)
@Grapes([
    @Grab(group='org.apache.tika', module='tika-core', version='1.14'),
    @Grab(group='commons-io', module='commons-io', version='2.5'),
    @Grab(group='commons-collections', module='commons-collections', version='2.1.1'),
    @Grab(group='org.openimaj', module='faces', version='1.3.5')
])

import static groovy.io.FileType.FILES

import javax.imageio.ImageIO

import java.util.concurrent.Callable
import java.util.concurrent.Executors

import org.apache.tika.Tika

import org.openimaj.image.ImageUtilities
import org.openimaj.image.colour.Transforms
import org.openimaj.image.model.pixel.HistogramPixelModel
import org.openimaj.image.processing.face.detection.HaarCascadeDetector
import org.openimaj.image.processing.face.detection.keypoints.FKEFaceDetector

def executorService = Executors.newFixedThreadPool(24)

new File('/vol/data/assets').eachFileRecurse(FILES) { file ->

  executorService.submit({ ->

    def haarCascadeDetector = new HaarCascadeDetector(60)
    def fkeFaceDetector = new FKEFaceDetector()

    def acceptableImageTypes = ['image/jpeg', 'image/png']
    def tika = new Tika()

    def detectedType = tika.detect(file)

    if (acceptableImageTypes.contains(detectedType)) {

      def mbfImage = ImageUtilities.createMBFImage(ImageIO.read(file), true)
      def fImage = Transforms.calculateIntensity(mbfImage)
      def haarCascadeDetectorFaces = haarCascadeDetector.detectFaces(fImage)
      def fkeFaceDetectorFaces = fkeFaceDetector.detectFaces(fImage)

      println "${file.path} # haarCascade score ${haarCascadeDetectorFaces.size()}, fkeFace score ${fkeFaceDetectorFaces.size()}"
    }
  })
}
```
