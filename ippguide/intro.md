---
title: How to Use the Internet Printing Protocol
author: Michael R Sweet, Peter Zehler
copyright: Copyright © 2017-2018 by The Printer Working Group
...

Chapter 1: Introduction
=======================


What is IPP?
------------

The Internet Printing Protocol ("IPP") is a secure application level protocol
used for network printing.  The protocol allows a Client to inquire about
capabilities of and defaults for a Printer (supported media sizes, two-sided
printing, etc.), inquire about the state of the Printer (paper out/jam, low
ink/toner, etc.), submit files for printing, and inquire about and/or cancel
submitted print Jobs.  IPP is supported by all modern network printers and
replaces all legacy network protocols including port 9100 printing
and LPD/lpr.

IPP is widely implemented in software as well, including the following open
source projects:

- [CUPS](https://www.cups.org/)
- [Java IPP Client Implementation](https://code.google.com/archive/p/jspi/)
- [Javascript IPP Client Implementation](https://github.com/williamkapke/ipp)
- [Python IPP Client Implementation](http://www.pykota.com/software/pkipplib/)
- [PWG IPP Sample Code](https://istopwg.github.io/ippsample)


IPP Overview
------------

The IPP architecture defines an abstract, hierarchical data model that provides
information about the printing process and the print jobs and capabilities of
the printer.  Because the semantics of IPP are attached to this model, the
client (software) does not need to know the internal details of the printer
(hardware).

IPP uses HTTP as its transport protocol.  Each IPP request is a HTTP POST with
an IPP message (and print file, if any) in the request message body.  The
corresponding IPP response is returned in the POST response message body.  The
IPP message itself uses a simple binary encoding that is described in the
[next section](#ipp-message-encoding).  HTTP connections can be unencrypted,
upgraded to TLS encryption using an HTTP OPTIONS request, or encrypted
immediately (HTTPS).  HTTP POST requests can also be authenticated using any of
the usual HTTP mechanisms like Basic (username and password).

> Note: Legacy network protocols do not support authentication, authorization,
> or privacy (encryption).

Printers are identified using Universal Resource Identifiers ("URIs") with the
"ipp" or "ipps" scheme, for example:

    ipp://printer.example.com/ipp/print
    ipps://printer2.example.com:443/ipp/print
    ipps://server.example.com/ipp/print/printer3

These are mapped to "http" and "https" URLs, with a default port number of 631
for IPP.  For example, the previous IPP URIs would be mapped to:

    http://printer.example.com:631/ipp/print
    https://printer2.example.com/ipp/print
    https://server.example.com:631/ipp/print/printer3

> The resource path "/ipp/print" is commonly used by IPP printers, however there
> is no hard requirement to follow that convention and older IPP printers used
> a variety of different locations.  Consult your printer documentation or the
> printer's Bonjour registration information to determine the proper hostname,
> port number, and path to use for your printer.

Print jobs are identified using the printer's URI and a job number that is
unique to that printer.


IPP Message Encoding
--------------------

IPP messages use a common format for both requests (from the client to the
printer) and responses (from the printer to the client).  Each IPP message
starts with a version number (2.0 is the most common), an operation (request) or
status (response) code, a request number, and a list of attributes.  Attributes
are named and have strongly typed values like integers, keywords, names, and
URIs.  Attributes are also placed in groups according to their usage -
the operation group for attributes used for the operation request or response,
the job group for print job attributes, and so forth.

The first two attributes in an IPP message are always "attributes-charset",
which defines the character set to use for all name and text strings, and
"attributes-natural-language", which defines the default language ("en" for
English, "fr" for French, "ja" for Japanese, etc.) for those strings.

The next attributes in a request are the printer's URI ("printer-uri") and, if
the request is targeting a print job, the job's ID number ("job-id").

Most requests include the name of the user that is submitting the request
("requesting-user-name").

A request containing an attached print file includes the MIME media type for
the file ("document-format").  The media type is 'text/plain' for text files,
'image/jpeg' for JPEG files, 'application/pdf' for PDF files, etc.

The following example encodes a Print-Job request using the `ipptool` test file
format:

```
{
    VERSION 2.0
    OPERATION Print-Job
    REQUEST-ID 42

    GROUP operation-attributes-tag
    ATTR charset "attributes-charset" "utf-8"
    ATTR naturalLanguage "attributes-natural-language" "en"
    ATTR uri "printer-uri" "ipp://printer.example.com/ipp/print"
    ATTR name "requesting-user-name" "John Doe"
    ATTR mimeMediaType "document-format" "text/plain"
    FILE "testfile.txt"
}
```

The same request using the CUPS API would look like the following:

```
#include <cups/cups.h>

...

http_t *http;
ipp_t *request, *response;

http = httpConnect2("printer.example.com", 631, NULL, AF_UNSPEC,
                    HTTP_ENCRYPTION_IF_REQUESTED, 1, 30000, NULL);

request = ippNewRequest(IPP_OP_PRINT_JOB);
ippAddString(request, IPP_TAG_OPERATION, IPP_TAG_URI, "printer-uri", NULL,
             "ipp://printer.example.com/ipp/print");
ippAddString(request, IPP_TAG_OPERATION, IPP_TAG_NAME, "requesting-user-name",
             NULL, "John Doe");
ippAddString(request, IPP_TAG_OPERATION, IPP_TAG_MIMETYPE, "document-format",
             NULL, "text/plain");

response = cupsDoFileRequest(http, request, "/ipp/print", "testfile.txt");

ipp_attribute_t *attr;
const char *name;
char value[2048];

for (attr = ippFirstAttribute(response); attr; attr = ippNextAttribute(response))
{
  name = ippGetName(attr);

  if (name)
  {
    ippAttributeString(attr, name, sizeof(name));
    printf("%s=%s\n", name, value);
  }
}
```

And this is how you'd send a Print-Job request using the nodejs API:

```
var ipp = require("ipp");
var printer = ipp.Printer("http://printer.example.com:631/ipp/print");
var fs = require("fs");
var document;

fs.readFile("testfile.txt", function(err, data) {
  if (err) throw err;

  document = data;
});

var msg = {
  "operation-attributes-tag": {
    "requesting-user-name": "John Doe",
    "document-format": "text/plain"
  },
  data: document;
};

printer.execute("Print-Job", msg, function(err, res) {
        console.log(err);
        console.log(res);
});
```

The response message uses the same version number, request number, character
set, and natural language values as the request.  A status code replaces the
operation code in the initial message header - for the Print-Job operation the
printer will return the 'successful-ok' status code if the print request is
successful or 'server-error-printer-busy' if the printer is busy and wants you
to try again at a later time.

The character set and natural language values in the response are followed by
operation-specific attributes.  For example, the Print-Job operation returns the
print job identifier ("job-id") and state ("job-state" and "job-state-reasons")
attributes.

> You can learn more about the IPP message encoding by reading the
> [Internet Printing Protocol/1.1: Encoding and Transport](https://tools.ietf.org/html/rfc8010)
> document.


Operations
----------

There are literally dozens of operations defined for IPP.  The following are
the most commonly used and are described in the following chapters:

- Create-Job: Create a new (empty) print job.

- Send-Document: Add a document to a print job.

- Print-Job: Create a new print job with a single document.

- Get-Printer-Attributes: Get printer status and capabilities.

- Get-Jobs: Get a list of queued jobs.

- Get-Job-Attributes: Get job status and options.

- Cancel-Job: Cancel a queued job.

Clients typically use the Create-Job and Send-Document operations to submit
files for printing so that these print jobs can be canceled while the document
data is being transferred to the printer.


Summary
-------

IPP is a widely implemented and secure application level protocol used for
network printing.  IPP defines an abstract model for printing that allows a
generic client application to communicate with any type of printer.  IPP uses
HTTP or HTTPS POST requests with binary encoded messages to perform a variety of
printing functions.  IPP supports multiple languages and file formats.
