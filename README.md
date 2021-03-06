README
======

– Created: April 2011/Last Revised: April 2013

I have been working through using [tesseract] (http://code.google.com/p/tesseract-ocr/)
for OCR processing, with the very handy [python-tesseract] 
(http://code.google.com/p/python-tesseract/) as a model for pulling it all together.

The main target has been newspaper images, which seems to stretch the limits of 
OCR software and there is some patching to increase parameters that
were problematic for this kind of material, as well as to add some addition
functionality.

The links above include the documentation for the SVN and development
versions of the application. The patches that are in use now are:

_for tesseract_
*   baseapi.h.patch
*   baseapi.cpp.patch
*   capi.h.patch
*   capi.cpp.patch

Assuming you have the svn version of tesseract (last used on tesseract 
version 866), patching would be a process like:

_for tesseract_

```
patch -p1 api/baseapi.h < /patches/baseapi.h.patch
patch -p1 api/baseapi.cpp < /patches/baseapi.cpp.patch
patch -p1 api/capi.cpp < /patches/capi.cpp.patch
patch -p1 api/capi.h < /patches/capi.h.patch
```

The original patches for python-tesseract are still included here but the
code used now is in the python-tesseract-mods directory. The "buildall"
and "cleanall" scripts create the "_tesseract.so" to be added to the
python library.

The main python application (ossocr.py) has several parameters for standalone
processing, but is set up by default for hadoop stream processing. For
standalone and testing, you probably want something like this:

```
python ossocr.py -c True -l eng -f page.jpg
```

where:

```
-c
    write coordinate information ("coords.xml" file is created)
-f
    specify input file, use for standalone, non-hadoop processing
-l
    tesseract language code
```

By default for standalone, the raw OCR goes to a file called "ocr.txt" 
but this can be changed with the -o parameter.

For a large number of files, hadoop is a good option for enlisting
multiple machines to provide better throughput:

```
hadoop jar /hadoop/contrib/streaming/hadoop-*streaming*.jar \
-file ./ossocr.py \
-mapper ./ossocr.py \
-file ./reducer.py \
-reducer ./reducer.py \
-input /home/hduser/files.txt \
-output /home/hduser/ossocr-output
```

You can test streaming from the command line:

```
cat files.txt | python ossocr.py
cat files.txt | python ossocr.py | python reducer.py
```

To allow some flexibility with files ramped up for OCR processing, a list of
filenames is streamed in ("files.txt" in the example above). This is done for
two reasons. One is that streaming in binary image data is tricky and this 
also allows files to be assembled in a temp directory, a much needed workaround
for processing images in the lab that I had access to for working through hadoop.

The other reason is that files can be specified with an "http" or "https"
prefix, which means they can reside ourside of local storage for processing. When
using Amazon Elastic MapReduce, this gives some flexibility for storing files
outside of Amazon's S3 service.

The "sortout.py" script will take the results and create XML file with
the coordinates of each word, and, optionally, a box file for coordinates as well 
as an hOCR file. The pixel gaps to define a new column and line can be set as 
parameters.

Like python-tesseract, php-tesseract uses [swig] (http://www.swig.org/) to
expose tesseract functions to php. The buildall script will 
try to create all of the components but the resulting library
(tesseract.so) will need to be referenced in the appropriate php.ini
file.

The "test.php" shows how the tesseract functions are invoked, for
example:

```
$mImgFile = "eurotext.tif";
$result=tesseract::ProcessPagesWrapper($mImgFile,$api);
printf("%s\n",$result);

$lenresult = tesseract::ExtractResultsWrapper($api, "coords.txt", strlen($result), "");
```

The php option is useful for Drupal environments and any of the many other spaces
where php is the hammer of choice.

It is still unclear what the optimum image processing should be for the most 
effective OCR, I have added some box support for dealing with what may be the 
most horrific example, which is newspaper scanned from microfilm and fiche. Tesseract 
seems to produce better results when a newspaper page is broken up into 
columns/sections, and then having each processed independently. 

I have used the [Line Segment Detector] 
(http://www.ipol.im/pub/algo/gjmr_line_segment_detector/)
to identify "lines" in a page, and then cut up the page based on that. This is 
tricky with microform images since they are often skewed, but it generally leads to 
better OCR. The Detector is typically invoked as a command line:

```
lsd newspaper_microfilm.pgm newspaper_microfilm.txt
```

where the results are placed in "newspaper_microfilm.txt" in this example. In 
hadoop mode, the ossocr.py script will look for a file with a ".txt" extension 
and try to use it to determine lines and squares on a page. Otherwise, there are 
several parameters for lines support. There are other line detection options, 
including within the tesseract code base, but the Line Segment Detector is usually 
quite effective for almost any image that uses lines for column identification. 

The "wcols.py" script is a simple attempt to identify columns that are not
based on lines, like the line segment application described below. The results
are disappointing for slanted images, but I include the script in case it
might be useful to someone else.

The most impressive image manipulation is available via the [Olena
project] (http://www.lrde.epita.fr/cgi-bin/twiki/view/Olena/WebHome), there
is now support for utilizing Olena via the [PAGE format] 
(http://schema.primaresearch.org/PAGE/gts/pagecontent/2010-03-19/) which
Olena produces. Like using the Line Segment Detector, the ossocr.py script
will look for a file with a ".xml" extension.

The "python-olena-hdlac" directory contains a swig project for using the olena
hdlac tool, this is really useful for newspaper scans.

The success of the OCR depends almost entirely on the image being processed. In
general, the contrast between the text and the background media seem to be the key
parameters, scanning pages directly from analogue sources may not require much
manipulation.

This is a work in progress, comments and suggestions welcome. 

art rhyno [Project Conifer] (http://projectconifer.ca)
