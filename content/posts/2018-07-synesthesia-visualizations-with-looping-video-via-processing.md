+++
title = "Synesthesia-powered visualizations with looping video via Processing"
description = "Processing powered video injection via Syphon in Synesthesia.live"
date = "2018-07-29"
tags = ["processing", "visualizations", "synesthesia"]
+++

For the last few months I've been on a quest to find pseudo algorithmic visualization software with support for realtime reaction to audio signals
and various levels of video support. Most of my interest in this is personal or artistic, for use at private gatherings with friends and what not.
That said, after spending a few late evenings searching using various keyword combinations, I began to take note of a few VJ software packages.
The VJ software wasn't entirely what I was looking for, but it did inform me so I could further query for and ultimately find the [Synesthesia](http://synesthesia.live) project
and wow, all I can say is 'ultra wow'!

In a nutshell, my words, Synesthesia is a framework that provides beat detection, accelerated rendering, video input, integration with Syphon, integration
with MIDI devices, and integration with a few other kits/frameworks I'm unfamiliar with. As an artist engineer, it is super simple
to write the JSON and glsl files (C, basically) required to render frames. The JSON is used to create the panes in the interface specific to each scene and
the glsl allows for the injection and reaction to beats, bass hits, treble, etc...

The core maintainers of the software are highly reactive via email and super supportive. This came in useful when it came time for me to build playlists of
Scenes along with different videos that I had referenced within Synesthesia. In it's current state, on Mac at least, Synesthesia doesn't
support video playlists along with the Scene playlists. Synesthesia supports taking video from a camera, video from a file (like mp4), or video from Syphon and using
that single video stream as it loops through its scenes (with their corresponding shaders). One of the maintainers informed me that support
for video playlists was near on the horizon, but I wanted something a bit sooner...

### Processing as a Syphon Video Server

Since Synesthesia takes Syphon as a video input, I began looking for Syphon video servers. I found a bunch of other expensive tools, VJ and otherwise.
Many of the tools I discovered are clearly made for niche, paid event, party production agencies based on their hefty price tags.

Lucky for me I came to realize that [Processing](https://processing.org/), an app I'm familiar with via some other shader work and my engagements with [baaahs](http://www.baaahs.org), has
a well written and maintained [Syphon plugin](https://github.com/Syphon/Processing) that wraps [JSyphon](https://github.com/Syphon/Java/tree/master/JSyphon). With this, I was able to whip up some night before the get-together code enabling
streaming videos and VJ loops into Synesthesia for processing. Yippie!

The code is Java-ish, as all things are in Processing and is pretty straight forward. The Processing project [SyphonVideoStreamer](https://github.com/joshdurbin/processing-syphon-video-player/blob/master/SyphonVideoStreamer/SyphonVideoStreamer.pde)
takes a pre-configured directory, scans it for `mp4|webm|mov` files, and randomly selects one of them to play. The selection is done
by randomly shuffling the set of videos and dropping them in queue, where they're pulled and played. There's also a target
runtime for VJ loops that are very short. If a video length is less than the target runtime, it will be looped  until it meets or exceeds the runtime.
Videos longer than the runtime will run just once before the next selection is made.

In addition to this, Synesthesia evaluates keyboard input for next video (n) and empty queue (e) functions allowing a "producer" to cycle back to a preferred
video without horribly disrupting the flow of things.

Make sure the Processing plugin "Syphon", version 2, is installed and you've set the video scan directory to your video(s) location prior to
running the project.

```java
import processing.video.Movie;
import java.util.List;
import codeanticode.syphon.SyphonServer;
import java.util.Queue;
import java.io.FilenameFilter;
import java.util.LinkedList;
import java.util.Collections;

SyphonServer server;
Movie currentMovie;
Integer innerVideoLoopCount;
Integer innerVideoLoopTarget;

final static Integer TARGET_PLAYTIME = 45;

final static String FLAT_MOVIE_DIRECTORY_ROOT = "/Users/jdurbin/Movies/VJ Loops";

static final FilenameFilter VIDEO_FILTER = new FilenameFilter() {
  final String[] EXTS = {
    "mp4", "webm", "mov"
  };

  boolean accept(File f, String name) {
    name = name.toLowerCase();
    for (String s : EXTS)  if (name.endsWith(s))  return true;
    return false;
  }
};

final List<Movie> movie = populateMovies();
final Queue<Movie> queuedMovies = new LinkedList<Movie>();

void setup() {

  server = new SyphonServer(this, "Synesthesia Video Streamer");
  size(1280, 720, P2D);
  currentMovie = getNextMovie();
  currentMovie.play();
  currentMovie.volume(0);

  innerVideoLoopCount = 0;
  innerVideoLoopTarget = Math.round(TARGET_PLAYTIME / currentMovie.duration());
  currentMovie.play();
  currentMovie.volume(0);
}

Movie getNextMovie() {

  if (queuedMovies.isEmpty()) {
    Collections.shuffle(movie);
    queuedMovies.addAll(movie);
  }

  return queuedMovies.poll();
}

List<Movie> populateMovies() {

  final List<Movie> videos = new ArrayList<Movie>();
  final File movieDirectory = dataFile(FLAT_MOVIE_DIRECTORY_ROOT);

  for (String moviePath : movieDirectory.list(VIDEO_FILTER)) {
    videos.add(new Movie(this, FLAT_MOVIE_DIRECTORY_ROOT + "/" + moviePath));
  }

  return videos;
}

void draw(){

  image(currentMovie, 0, 0);
  set(width - currentMovie.width >> 1, height - currentMovie.height >> 1, currentMovie);
  server.sendScreen();

  if (currentMovie.duration() == currentMovie.time() && innerVideoLoopCount == innerVideoLoopTarget) {

    currentMovie.stop();
    currentMovie = getNextMovie();
    currentMovie.play();
    currentMovie.volume(0);

    innerVideoLoopCount = 0;
    innerVideoLoopTarget = Math.round(TARGET_PLAYTIME / currentMovie.duration());

  } else if (currentMovie.duration() == currentMovie.time() && innerVideoLoopCount < innerVideoLoopTarget) {

    innerVideoLoopCount++;

    currentMovie.stop();
    currentMovie.play();
    currentMovie.volume(0);
  }
}

void movieEvent(Movie m) {
  m.read();
}

void keyPressed() {

  if (key == 'n') {

    currentMovie.stop();
    currentMovie = getNextMovie();
    currentMovie.play();
    currentMovie.volume(0);
  } else if (key == 'e') {

    currentMovie.stop();
    queuedMovies.clear();
    currentMovie = getNextMovie();
    currentMovie.play();
    currentMovie.volume(0);
  }

}
```

### Integration with Synesthesia

Once the project is up and running we can bounce over the Synesthesia to connect Processing-based Syphon input source:

![image](/img/2018-07-synesthesia-visualizations-with-looping-video-via-processing/syn_setup1.png)

...then in our scene configuration set the controls to use the Syphon video source:

![image](/img/2018-07-synesthesia-visualizations-with-looping-video-via-processing/syn_setup2.png)

Enjoy!

