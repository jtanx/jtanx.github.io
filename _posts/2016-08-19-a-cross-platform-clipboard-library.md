---
layout: post
title: Writing a cross-platform clipboard library
---

While working on FontForge to port it away from its hard X11 dependency, something that surprised me was the apparent lack of any *decent* cross - platform clipboard library. Searching for what was out there, the answer was either *"use Qt"*, *"use Gtk"*, or just *"write your own"*. Indeed, using a cross - platform toolkit is the way to go 90% of the time, but at times, you just want something simple and without the extra baggage.

What I wanted was a self - contained library that worked on all three (Windows, Linux, OS X) platforms, and more importantly, had a pure C API. Existing options ranged from only supporting two out of the three platforms (OS X and Windows, or Windows and Linux), to having half - baked solutions that called `xclip` as its Linux 'implementation'.

So in answer to that, I developed [libclipboard](https://github.com/jtanx/libclipboard).

<!--more-->

# Contents
{: .no_toc}

* TOC
{: toc}

---- -

At present, only UTF-8 text copy/pasting is supported. This seems fine for an initial release - I may want to add image support later, but that walks into a whole other minefield of what constitutes a 'standard' raw image format.

## The API
Right off the bat, I wanted a simple API. At its core, the API can be summed up as:

`````c
clipboard_c *cb = clipboard_new(NULL);
char *some_text_from_clipboard = clipboard_text(cb);
clipboard_set_text(cb, "I put some text on the clipboard!");
clipboard_clear(cb, LCB_CLIPBOARD);
clipboard_free(cb);
`````

At the same time, some extra control would be nice, such as to get back the text length, or to set to *which* clipboard you want to copy to (on Linux). So in true WinAPI fashion, there are extended versions:

`````c
char *some_text_from_clipboard = clipboard_text_ex(cb, &length, LCB_PRIMARY);
int max_length = 10;
clipboard_set_text_ex(cb, some_long_string, max_length, LCB_PRIMARY);
`````

At present, for simplicity, the API is completely synchronous. This isn't an issue on Windows and OS X, where the backing implementation is synchronous anyway, but on Linux, it effectively hides the asynchronous nature of X11.

Some less important functions include `clipboard_has_ownership`. The intention of this function was to return a value that indicated whether or not the current instance 'owned' what was presently on the clipboard. In practice, this is less useful, because it's subject to race conditions - at any time, ownership could change if another user decides to paste new data onto the clipboard.

## Implementation details

### Windows
On Windows, there exists a fairly nice [clipboard API](https://msdn.microsoft.com/en-us/library/windows/desktop/ms649016(v=vs.85).aspx), so the implementation essentially boils down to taking the UTF-8 string, performing a wide character conversion, and sticking that on the clipboard, with the reverse happening on retrieval.

Note that as Windows only has the concept of a global clipboard, attempting to read data from it may fail if someone else has a lock on the clipboard at the time of request. Currently, libclipboard attempts something like 5 retries to obtain the lock with a waiting time of around 5 milliseconds between retries. If this still fails, then the relevant failure value is returned by the API (`false` for `clipboard_set_text` and `NULL` for `clipboard_text`).

There are no external dependencies for this library on Windows.

### Linux
In terms of simplicity, the same simply *cannot* be said for Linux. For such a simple concept as a clipboard, you have to jump through a *series* of hoops to get a working implementation - by far, the Linux implementation is the most complex. The clipboard on Linux relies on the X11 concept of selections as a generic method to send arbitrary data between program instances. While the concept of selections is nice, its generality ends up also being a *huge* drawback when all you want to do is copy and paste things onto the clipboard. This is also not to mention the fact that X11 selections is only one (if not the most prevalent) way. With the introduction of the Wayland protocol, there is *yet another* method in which to send clipboard data. libclipboard does not implement Wayland support as of yet (and likely won't for the foreseeable future!).

The gist of how the clipboard works on X11 is that a window must claim ownership of a given selection, and then advertise available formats of data to whomever wishes said data. Once advertised, the requestor then *requests* the data from the owner, who then through X11, transmits the requested data. I am led to believe that back in the good old days, there was no standardised format for which to send clipboard data as. Fortunately, this has now been mostly standardised through [ICCCM](https://en.wikipedia.org/wiki/Inter-Client_Communication_Conventions_Manual). Note that libclipboard implements some of ICCCM, but not all of it. In particular, it doesn't support the MULTIPLE atom, nor the INCR method of sending/receiving chunked data.

Implementation - wise, all of this means that libclipboard depends on XCB (libxcb) and pthreads for its Linux/X11 implementation. A message - only X11 window is created, and its event loop is then run in its own thread. When text is 'pasted', the window claims ownership of the relevant selection and sends the text to any other window that requests it. When text needs to be retrieved from the clipboard, a request is made to the window owning the relevant selection before waiting for the reply containing the text. Of course, if we already own the selection, talking with the X11 server is skipped altogether and a copy of the data is returned directly to the requestor.

In reality, there were two choices of libraries that I could have used - either Xlib or XCB. Xlib is the grand - daddy of X11 implementations, with XCB (literally the X - protocol C - language binding) being the slightly newer variant. Initially, I was going to use Xlib, but instead used XCB, which decidedly has a cleaner API. XCB also has better threading support, which was essential in this case, because of the event loop running in its own thread.

Note that this implementation means that it's subject to request timeouts - in particular, an abnormally behaving server or window could potentially not respond to our request for a selection. By default, libclipboard waits 1.5 seconds for a response before timing out. This setting can be configured when the clipboard context is created.

Also of note is the fact that **your program must remain running so long as you want your data to be available on the clipboard**. Okay, so this is not strictly true if you're also running a clipboard manager such as Klipper, but this is something that you must be aware of when using this implementation.

### OS X
Funnily enough, the platform I had least experience with, OS X, also had the simplest implementation of them all. This implementation is written in Objective C, and uses the [NSPasteboard](https://developer.apple.com/library/mac/documentation/Cocoa/Reference/ApplicationKit/Classes/NSPasteboard_Class/) class (as part of the Cocoa/AppKit framework), with data being copied/pasted from/to the general pasteboard.

As with Windows, the backend implementation is inherently synchronous, so there are no caveats to look out for. Note however, that this is also the least tested implementation, simply because I have little experience with OS X. I also dislike Objective C.

## Unit testing
This library is *fairly* well unit tested, but probably not extensive as it should be. Unit tests are written in C++ using the [Google Test](https://github.com/google/googletest) framework. After looking at the unit testing libraries available in C, they simply didn't compare to the simplicity of using Google Test.

Note that the unit tests can fail sporadically on Linux - in particular on KDE if you're using Klipper with the option 'Prevent empty clipboard'.