---
author: doug
comments: true
date: 2014-04-16 16:54:30+00:00
layout: post
slug: decompressing-gzipped-with-javascript
title: Decompressing gzipped data with Javascript
wordpress_id: 314
---

I found myself in a situation where I needed to work with gzipped files with javascript. In my case, they were gzipped ASCII text. I couldn't find an exact recipe, however, I did find a [really nice tip on stack exchange](http://stackoverflow.com/questions/14727856/fetching-zipped-text-file-and-unzipping-in-client-browers-feasible-in-javascrip), which primarily led me to [Mozilla's javascript docs about working with binary data](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/Sending_and_Receiving_Binary_Data). This relies on the [imaya/zlib.js](https://github.com/imaya/zlib.js) project, which has a nicely packed up gunzip module.

Additionally, [I've got it posted on PasteAll.](http://www.pasteall.org/50883/javascript) (which is a wicked nice paste bin if you haven't used it, simple, pretty, and run by a really good OSS guy.)

```javascript
function loadCompressedASCIIFile(request_url) {

    var req = new XMLHttpRequest();

    // You gotta trick it into downloading binary.
    req.open('GET', request_url, false);
    req.overrideMimeType('text\/plain; charset=x-user-defined');    
    req.send(null);

    // Check for any error....
    if (req.status != 200) {
        return '';
    }

    // Here's our raw binary.
    var rawfile = req.responseText;

    // Ok you gotta walk all the characters here
    // this is to remove the high-order values.

    // Create a byte array.
    var bytes = [];

    // Walk through each character in the stream.
    for (var fileidx = 0; fileidx < rawfile.length; fileidx++) {
        var abyte = rawfile.charCodeAt(fileidx) & 0xff;
        bytes.push(abyte);
    }

    // Instantiate our zlib object, and gunzip it.    
    // Requires: http://goo.gl/PIqhbC [github]
    // (remove the map instruction at the very end.)
    var  gunzip  =  new  Zlib.Gunzip ( bytes ); 
    var  plain  =  gunzip.decompress ();

    // Now go ahead and create an ascii string from all those bytes.
    var asciistring = "";
    for (var i = 0; i < plain.length; i++) {         
         asciistring += String.fromCharCode(plain[i]);
    }

    return asciistring;

}
```




