---
layout: post
title: Symbolicate macOS (and iOS) apps crash reports on Linux
type: post
comments: true
categories:
- ios
tags:
- ios
- macos
- crash
- crashlog
- ips
- lldb
- atos
author: Dominik Kapusta
permalink: "/2022/06/13/symbolicate-macos-crash-reports-on-linux"
redirect_from:
  - /2022/06/13/symbolicate-macos-crashes-on-linux
excerpt: If you use a custom (in-house?) crashlog collection service and
  want to symbolicate crashlogs server-side, you don't need a macOS machine.
---

If you need to symbolicate crash reports for your macOS or iOS app yourself,
you'd probably use `atos` command line utility to get the job done. Apple has
in-depth [documentation](https://developer.apple.com/documentation/xcode/adding-identifiable-symbol-names-to-a-crash-report)
on crash reports symbolication, and they recommend `atos` in the section called
[Symbolicate the Crash Report with the Command Line](https://developer.apple.com/documentation/xcode/adding-identifiable-symbol-names-to-a-crash-report#Symbolicate-the-Crash-Report-with-the-Command-Line).

Basically what `atos` does is:
* take the dSYM file, the architecture name and the load address of the app's binary image,
* take the list of memory addresses to symbolicate,
* return the list of symbols identifying the memory addresses in question.

If you ever wanted to automate symbolication (not relying on a 3rd party
service) you might have noticed that `atos` is only available on macOS.
This limits your options for performing symbolication server-side, requiring
you to provision a Mac server – and these machines tend to be expensive and/or cumbersome to maintain.

However, it appears that you still can symbolicate an Apple-platform crash report
on Linux. It only takes a bit more [Command Line Fu](https://xkcd.com/196/).

# Sample app

In this article we'll work with a sample application that crashes. It contains
just a single window and a button. Tapping the button would attempt to update
a non-existent text field:

```swift
class ViewController: NSViewController {
    @inline(never)
    @IBAction func buttonAction(_ sender: Any?) {
        updateLabel()
    }

    @inline(never)
    private func updateLabel() {
        label.stringValue = "Boom"
    }

    private weak var label: NSTextField!
}
```

Note that `@inline(never)` attribute is there to instruct the compiler to not try and
inline the functions (collapsing the stack trace), which would make this example harder
to follow. It should not be used here in a real-life scenario.

A sample crash stack trace (edited for brevity) looks like this:

```
Thread 0 Crashed::  Dispatch queue: com.apple.main-thread
0   SampleCrashingApp        0x1004aeb3c 0x1004ac000 + 11068
1   SampleCrashingApp        0x1004aeaf4 0x1004ac000 + 10996
2   SampleCrashingApp        0x1004aeab8 0x1004ac000 + 10936
3   AppKit                   0x185c9bbc4 -[NSApplication(NSResponder) sendAction:to:from:] + 460
    [... more AppKit frames]
16  AppKit                   0x185ec0848 -[NSApplication _handleEvent:] + 76
17  AppKit                   0x185a88618 -[NSApplication run] + 636
18  AppKit                   0x185a59d08 NSApplicationMain + 1132
19  SampleCrashingApp        0x1004aecf4 0x1004ac000 + 11508
20  dyld                     0x1006c1088 start + 516
```

# Environment setup

In the absence of `atos` on Linux, you'll need the following tools to get the job done:
* _llvm-nm_: to find the base image address as stored in the dSYM file,
* _llvm-addr2line_: to map symbol addresses to specific lines of code,
* _swift-demangle_: to convert mangled Swift symbol names to a human-readable form.

The first two are included in the `llvm` package available in main package tree on most
Linux distributions, including Debian-based (e.g. Ubuntu) and RHEL-based (e.g. Amazon Linux).
The third one requires Swift to be installed in the system. Swift, in turn, is likely
unavailable from your preferred Linux distro's package manager, so you'd have to install
it manually or use a Docker image (as listed in the [Downloads](https://www.swift.org/download/) page at swift.org).

Apart from the tools, you'll need your crash report and your app's debug symbols (as a dSYM bundle).

# Symbolication

## Note: always specify architecture

Please note that you must observe the architecture specified in the crash report (x86_64 vs arm64)
and pass it over to symbolication-related commands. This is because app (and dSYMs) are fat
binaries containing sets of binary code supporting specific architectures. The debug symbol addresses
will differ between architectures.

## Get the binary image load address

Unsymbolicated stack frames are described using binary image load address and offset, for instance:

```
1   SampleCrashingApp        0x1004aeaf4 0x1004ac000 + 10996
```
Let's have a look at the numbers on the right hand side:
* `0x1004aeaf4` is a symbol address,
* `0x1004ac000` is a binary image load address,
* `10996` is symbol's offset within a binary image.

The symbol address is a sum of the binary image load address and the symbol offset (hence the plus sign).
Note that the left operand is a hexadecimal number while the right operand is decimal... 🤯
The binary image load address varies between application sessions, and symbol offsets
are constant per binary image (i.e. an offset for a given symbol will be the same
across application sessions).

When using `atos` you need to pass the load address and the symbol address as parameters.
With `llvm-addr2line` it's a bit different:
1. You only pass the symbol address, without the load address.
2. The symbol address is different than the one specified in the crash report 💥.
  To be more specific, it's the load address part that differs. Instead of an
  arbitrary, session-dependent value, it seems to be fixed to `0x100000000`.

Arguably it makes more sense this way -- after all the tool operates on the dSYM file so
it looks up the same addresses for the same symbols all the time. Likely it's just a convenience
of `atos` that it works with addresses present in the crash report, and under the hood it
just subtracts load address from the symbol address and works on the resulting offset.

Anyways, the `0x100000000` magic number is actually present in the dSYM (and the app binary)
symbol table and can be extracted with the following command:

```bash
$ llvm-nm -P --arch=arm64 \
    SampleCrashingApp.app.dSYM/Contents/Resources/DWARF/SampleCrashingApp \
    | grep __mh_execute_header
```
The above would output:
```
__mh_execute_header T 100000000 0
```

## Resolve addresses to symbols

Knowing the normalized load address, we can proceed to computing symbol addresses
and resolving them. A little calculation is required here. Let's go back to our stack trace:

```
0   SampleCrashingApp        0x1004aeb3c 0x1004ac000 + 11068
1   SampleCrashingApp        0x1004aeaf4 0x1004ac000 + 10996
2   SampleCrashingApp        0x1004aeab8 0x1004ac000 + 10936
```

We don't care about addresses anymore, and are only interested in symbol offsets,
i.e. `11068`, `10996` and `10936` (decimals!). Those need to be added to `0x100000000`, making:
* `0x100000000` + `11068` = `0x100002b3c`
* `0x100000000` + `10996` = `0x100002af4`
* `0x100000000` + `10936` = `0x100002ab8`

These addresses can now be passed to `llvm-addr2line`:

```bash
$ llvm-addr2line -f \
    --default-arch=arm64 \
    --obj SampleCrashingApp.app.dSYM/Contents/Resources/DWARF/SampleCrashingApp \
    0x100002af4
```

This tool would output the source code location (file and line number) of the function
represented by the stack frame. Adding `-f` will make it also print the function name.
Here is a sample output (always 2 lines):

```
$s17SampleCrashingApp14ViewControllerC11updateLabel33_CEBA0EBCB7099BA3FFA1062F19F801EELLyyF
/Users/user/code/SampleCrashingApp/SampleCrashingApp/ViewController.swift:18
```

The top line is a mangled Swift symbol, so it needs to be demangled in a separate command:

```bash
$ swift-demangle --simplified --compact \
    '$s17SampleCrashingApp14ViewControllerC11updateLabel33_CEBA0EBCB7099BA3FFA1062F19F801EELLyyF'
```
rendering the actual function name:
```
ViewController.updateLabel()
```

## Output

After symbolicating all stack frames and gluing the data together, the stack trace
symbolicated using LLVM on Linux looks like this:

```
Thread 0 Crashed::  Dispatch queue: com.apple.main-thread
0   SampleCrashingApp        Swift runtime failure: Unexpectedly found nil while implicitly unwrapping an Optional value (ViewController.swift:0)
1   SampleCrashingApp        ViewController.updateLabel() (ViewController.swift:18)
2   SampleCrashingApp        @objc ViewController.buttonAction(_:) (<compiler-generated>:0)
```

And here is the same stack trace symbolicated on macOS using `atos`:

```
Thread 0 Crashed::  Dispatch queue: com.apple.main-thread
0   SampleCrashingApp        ViewController.updateLabel() (in SampleCrashingApp) (ViewController.swift:18)
1   SampleCrashingApp        ViewController.updateLabel() (in SampleCrashingApp) (ViewController.swift:18)
2   SampleCrashingApp        @objc ViewController.buttonAction(_:) (in SampleCrashingApp) (<compiler-generated>:0)
```

That's not bad already!

# Additional remarks

## llvm-symbolizer

This tool is shipped with LLVM alongside `llvm-addr2line` and it does roughly
the same -- takes similar arguments and produces similar output. I've only found
it recently and still need to learn the difference between these.
By quick checking it looks like `llvm-symbolizer` defaults to displaying inlined
symbols, and it also provides the source code column, in addition to the row.
And historically, `llvm-addr2line` was designed as a drop-in replacement to
GNU `addr2line`. Perhaps it's just a matter of preference to pick either of the two.

## Batch operations

Both `llvm-addr2line` and `llvm-symbolizer` can take multiple symbols for processing.

## Sample Dockerfile

This Dockerfile contains all the software required to symbolicate Apple
crash reports on Linux:

```Dockerfile
FROM swiftlang/swift:nightly-5.6-amazonlinux2
RUN yum -y update && yum -y install llvm
```

It uses one of the [official Swift Docker images](https://hub.docker.com/r/swiftlang/swift).


<div id="disqus_thread"></div>
<script>

/**
*  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
*  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables*/

var disqus_config = function () {
this.page.url = 'https://kapusta.cc/2022/06/13/symbolicate-macos-crashes-on-linux';  // Replace PAGE_URL with your page's canonical URL variable
this.page.identifier = '2022-06-13-symbolicate-macos-crashes-on-linux'; // Replace PAGE_IDENTIFIER with your page's unique identifier variable
};

(function() { // DON'T EDIT BELOW THIS LINE
var d = document, s = d.createElement('script');
s.src = 'https://kapusta-cc.disqus.com/embed.js';
s.setAttribute('data-timestamp', +new Date());
(d.head || d.body).appendChild(s);
})();
</script>
<noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>

