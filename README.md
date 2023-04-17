# Web Printing Explainer

## Introduction

The `navigator.printing` Web API is a proposed standard for enabling printer-related functionality in web applications. Built on top of the Internet Printing Protocol (IPP), this API provides a set of JavaScript methods that allow developers to query local printers, submit print jobs, and manage print job options and status directly from web applications.

Printing is a common and important task in many domains, including document management, label printing, receipt printing, and more. While web applications can generate printable content such as PDFs, images, and text documents, sending print jobs from web applications to local printers has traditionally required various workarounds such as relying on third-party plugins or polyfilling via `window.print()`; `navigator.printing` API aims to provide a standardized and seamless way for web applications to directly interact with printers, enabling developers to build printer-related features within their web applications.

The explainer outlines the motivating use cases for the API, its key features, and how it can benefit web developers and users alike.

## Motivating Use Cases

### Augmenting `window.print()`

While `window.print()` is a basic function that triggers the browser's default print dialog, the `navigator.printing` API provides a more comprehensive set of methods and options for querying local printers, submitting print jobs with customization options, managing print jobs, and customizing print workflows. This extended functionality allows developers to have more control over the printing process and enables advanced use cases, such as printing specific pages or sections of a document, setting print options like paper size and orientation, and monitoring the status of print jobs.

### Leveraging Progressive Web Apps
The new Web API can motivate developers to switch to Progressive Web Apps (PWAs) by offering enhanced printing capabilities that were traditionally limited to native applications.

## Key Features
The API primarily focuses on listing local printers and their capabilities and sending print jobs to them.
Support for other IPP capabilities as well as exhaustive enumeration of all possible attributes is out of the scope of this proposal.

Three new interfaces are introduced as part of the API:
- The `Printing` interface is a singleton that can be accessed as `navigator.printing`.
  - `Promise<sequence<Printers>> navigator.printing.getPrinters()` enumerates local printers and returns a list of `Printer` objects.
- The `Printer` interface provides a way to interact with printers: retrieve their properties and capabilities, initiate print jobs, and monitor printer state changes.
  - `Promise<PrinterAttributes> getPrinterAttributes()` implements the `Get-Printer-Attributes` IPP attributes and returns a dictionary with a subset of supported printer attributes specified in section [5.4 of RFC8011](https://www.rfc-editor.org/rfc/rfc8011#section-5.4).
  - `Promise<PrintJob> printJob(...)` implements the `Print-Job` IPP attribute and allows developers to customize the print job via a subset of supported job template attributes specified in section [5.2 of RFC8011](https://www.rfc-editor.org/rfc/rfc8011#section-5.2).
  - `readonly attribute PrinterState state` and `attribute EventHandler onstatechange` allows developers to track the printer state and wait until it becomes ready to accept print jobs.
- The `PrintJob` interface provides a way to interact with print jobs: monitor their state changes and cancel on demand.
  - `void cancel()` can be invoked to attempt to cancel a print job.
  - `readonly attribute PrintJobState state` and `attribute EventHandler onstatechange` allows developers to track the job state and wait for its completion.

## Usage Examples

### Listing Printers
```js
try {
  const printers = await navigator.printing.getPrinters();
  printers.forEach(printer => {
    printer.getPrinterAttributes().then(attributes => {
      console.log(`${printer.name} has the following attributes: ${attributes}`);
    });
  });
} catch (err) {
  console.warn("Printing operation failed: " + err);
}
```

### Tracking Printer State
```js
try {
  const printers = await navigator.printing.getPrinters();
  const printer = printers.find(printer => printer.name === 'Brother QL-820NWB');
  printer.onstatechange = () => {
    console.log(`${printer.name}'s new state is ${printer.state}!`);
  };
} catch (err) {
  console.warn("Printing operation failed: " + err);
}
```

### Submitting a Print Job
```js
try {
  const printers = await navigator.printing.getPrinters();
  const printer = printers.find(printer => printer.name === 'Brother QL-820NWB');

  const printJob = await printer.printJob("Sample Print Job", {
    data: new Blob(...),
    'document-format': 'application/pdf',
  }, {
    copies: 2,
    media: 'iso_a4_210x297mm',
    'multiple-document-handling': 'separate-documents-collated-copies',
    'printer-resolution': {
      x: 300,
      y: 400,
      units: ResolutionUnits.DOTS_PER_INCH,
    },
    sides: 'one-sided',
    'print-quality': PrintQuality.BEST_QUALITY,
    'page-ranges': [{from: 1, to: 5}, {from: 7, to: 10}],
  });

  const printJobComplete = new Promise((resolve, reject) => {
    printJob.onstatechange = () => {
      if (IsErrorStatus(printJob.state)) {
        console.warn(`Job errored: ${printJob.state}`);
        reject(/**/);
        return;
      }
      if (printJob.state === PrintJobState.COMPLETED) {
        console.log("Job complete!");
        resolve(/**/);
        return;
      }
      console.log(`Job state changed to ${printJob.state}`);
    };
  });
  await printJobComplete;
} catch (err) {
  console.warn("Printing operation failed: " + err);
}
```

### Canceling a Print Job
```js
try {
  const printers = await navigator.printing.getPrinters();
  const printer = printers.find(printer => printer.name === 'Brother QL-820NWB');

  const printJob = await printer.printJob(...);

  // This might take no effect if the job has already finished.
  printJob.cancel();

} catch (err) {
  console.warn("Printing operation failed: " + err);
}
```

## WebIDL
```cs
// enums

// https://www.rfc-editor.org/rfc/rfc8011#section-5.1.10
enum MimeMediaType {
  "application/pdf",
};

// https://www.rfc-editor.org/rfc/rfc8011#section-5.2.11
typedef DOMString Media;

// https://www.rfc-editor.org/rfc/rfc8011#section-5.2.4
enum MultipleDocumentHandling {
  "separate-documents-collated-copies",
  "separate-documents-uncollated-copies",
};

// https://www.rfc-editor.org/rfc/rfc8011#section-5.2.10
enum Orientation {
  PORTRAIT = 3,
  LANDSCAPE = 4,
  REVERSE_LANDSCAPE = 5,
  REVERSE_PORTRAIT = 6,
};

enum PrintColorMode {
  "monochrome",
  "color",
};

// https://www.rfc-editor.org/rfc/rfc8011#section-5.2.13
enum PrintQuality {
  DRAFT = 3,
  NORMAL = 4,
  BEST_QUALITY = 5,
};

enum ResolutionUnits {
  DOTS_PER_INCH = 3,
  DOTS_PER_CENTIMETER = 4,
};

enum Sides {
  "one-sided",
  "two-sided-long-edge",
  "two-sided-short-edge",
};

enum PrintJobState {
  // ...
};

enum PrinterState {
  // ...
};

// dictionaries

dictionary DocumentDescription {
  required Blob data;
  MimeMediaType document-format;
};

// https://www.rfc-editor.org/rfc/rfc8011#section-5.2.12
dictionary Resolution {
  unsigned long x;
  unsigned long y;
  ResolutionUnits units;
};

dictionary Range {
  unsigned short from;
  unsigned short to;
};

dictionary JobTemplateAttributes {
  // RFC8011, §5.2.5
  // https://www.rfc-editor.org/rfc/rfc8011#section-5.2.5
  unsigned short copies;
------------------------------------------------------
  // RFC8011, §5.2.11
  // https://www.rfc-editor.org/rfc/rfc8011#section-5.2.11
  Media media;
------------------------------------------------------
  // RFC8011, §5.2.4
  // https://www.rfc-editor.org/rfc/rfc8011#section-5.2.4
  MultipleDocumentHandling multiple-document-handling;
------------------------------------------------------
  // RFC8011, §5.2.10
  // https://www.rfc-editor.org/rfc/rfc8011#section-5.2.10
  Orientation orientation-requested;
------------------------------------------------------
  // RFC8011, §5.2.7
  // https://www.rfc-editor.org/rfc/rfc8011#section-5.2.7
  sequence<Range> page-ranges;
------------------------------------------------------
  // RFC8011, §5.2.12
  // https://www.rfc-editor.org/rfc/rfc8011#section-5.2.12
  Resolution printer-resolution;
------------------------------------------------------
  // PWG5100.13, §6.2.3
  // https://ftp.pwg.org/pub/pwg/candidates/cs-ippnodriver20-20230301-5100.13.pdf
  PrintColorMode print-color-mode;
------------------------------------------------------
  // RFC8011, §5.2.13
  // https://www.rfc-editor.org/rfc/rfc8011#section-5.2.13
  PrintQuality print-quality;
------------------------------------------------------
  // RFC8011, §5.2.8
  // https://www.rfc-editor.org/rfc/rfc8011#section-5.2.8
  Sides sides;
};

dictionary PrinterAttributes {
  // See |copies| in JobTemplateAttributes.
  unsigned short copies-default;
  Range copies-supported;
------------------------------------------------------
  // See |document-format| in DocumentDescription.
  // RFC8011, §5.4.21
  // https://www.rfc-editor.org/rfc/rfc8011.html#section-5.4.21
  MimeMediaType document-format-default;
  // RFC8011, §5.4.22
  // https://www.rfc-editor.org/rfc/rfc8011.html#section-5.4.22
  sequence<MimeMediaType> document-format-supported;
------------------------------------------------------
  // See |media| in JobTemplateAttributes.
  Media media-default;
  sequence<Media> media-supported;
------------------------------------------------------
  // See |multiple-document-handling| in JobTemplateAttributes.
  MultipleDocumentHandling multiple-document-handling-default;
  sequence<MultipleDocumentHandling> multiple-document-handling-supported;
------------------------------------------------------
  // See |orientation-requested| in JobTemplateAttributes.
  Orientation orientation-requested-default;
  sequence<Orientation> orientation-requested-supported;
------------------------------------------------------
  // See |page-ranges| in JobTemplateAttributes.
  boolean page-ranges-supported;
------------------------------------------------------
  // RFC8011, §5.4.6
  // https://www.rfc-editor.org/rfc/rfc8011.html#section-5.4.6
  DOMString printer-info;
  // RFC8011, §5.4.5
  // https://www.rfc-editor.org/rfc/rfc8011.html#section-5.4.5
  DOMString printer-location;
  // RFC8011, §5.4.9
  // https://www.rfc-editor.org/rfc/rfc8011.html#section-5.4.9
  DOMString printer-make-and-model;
------------------------------------------------------
  // See |printer-resolution| in JobTemplateAttributes.
  Resolution printer-resolution-default;
  sequence<Resolution> printer-resolution-supported;
------------------------------------------------------
  // See |print-color-mode| in JobTemplateAttributes.
  PrintColorMode print-color-mode-default;
  sequence<PrintColorMode> print-color-mode-supported;
------------------------------------------------------
  // See |print-quality| in JobTemplateAttributes.
  PrintQuality print-quality-default;
  sequence<PrintQuality> print-quality-supported;
------------------------------------------------------
  // See |sides| in JobTemplateAttributes.
  Sides sides-default;
  sequence<Sides> sides-supported;
};

// interfaces

typedef DOMString JobId;
typedef DOMString PrinterId;

interface PrintJob {
  readonly attribute JobId id;
  // User-supplied title from Printer.printJob()
  readonly attribute DOMString title;

  readonly attribute PrintJobState state;
  // Plain event
  attribute EventHandler onstatechanged;

  void cancel();
};

interface Printer {
  readonly attribute PrinterId id;

  // Corresponds to printer-name from RFC8011 §5.4.4
  // https://www.rfc-editor.org/rfc/rfc8011.html#section-5.4.4
  readonly attribute DOMString name;

  readonly attribute PrinterState state;
  // Plain event
  attribute EventHandler onstatechanged;

  Promise<PrinterAttributes> getPrinterAttributes(
    optional sequence<OptionalPrinterAttribute> optionalAttributes = []);

  Promise<PrintJob> printJob(
    DOMString title,
    DocumentDescription document,
    optional JobTemplateAttributes options = {});
};

[Exposed=Window, SecureContext]
interface Printing {
  Promise<Array<Printer>> getPrinters();
};

[Exposed=Window, SecureContext]
partial interface Navigator {
  [SameObject] readonly attribute Printing printing;
};
```

## Security & Privacy Considerations

The scenarios described here refer to API access by websites in general and can be read as a justification for the recommendation of having it available only to [Isolated Web Apps](https://github.com/reillyeon/isolated-web-apps/blob/main/README.md).

### Fingerprinting
The detailed information about the available local printers provide an [active fingerprinting](https://www.w3.org/TR/fingerprinting-guidance/#dfn-active-fingerprinting) surface. This is possible via the combination of `navigator.printing.getPrinters()` + `Printer.getPrinterAttributes()` API calls and does not require user consent. Some of these printer values are (but not limited to):
* printer-name
* printer-make-and-model
* printer-info
* printer-location

It's worth noting that printers are usually default-configured in private spaces and hence don't provide too much insight; however, the situation is often different in corporate environments where printers are often customly tailored by admins; thus any deviation from standard settings for a particular model might potentially give away a corporate user. Moreover, the same logic applies to printer tiers: the presence of an expensive enterprise-scale printer (as opposed to a common customer model) allows the website to accurately determine a business user.

A more intricate fingerprinting approach is to subscribe to `onstatechanged` notifications for a specific printer; by observing the patterns in printer state changes (i.e. how often it goes from `busy` to `idle` and vice versa) a macilious website could reveal sensitive information about the user's printing behavior. This information could be used to build a profile of the user or to track their activities across multiple sites.

#### Mitigation
A basic precaution is to introduce a new permissions-policy called `"printing"` to `navigator.printing.getPrinters()`  to facilitate protection against malicious third-party iframes.
A more sophisticated approach would include creating a permissions surface resembling those for camera/microphone and prompting the user whether an origin should be granted access to local printers.

### Forging Printer Jobs
A malicious website could use the printJob() method to create and send fake printer jobs to the user's printer, causing it to waste paper and ink or potentially print malicious content; ihis could even potentially happen with a legitimate website due an inaccuracy in client-side printer handling.

#### Mitigation
Submitting print jobs will require an explicit consent from the user similar to the system dialog shown during `window.print()`.

## Considered Alternatives

### Opaque Handles vs Interfaces
Instead of separating the `Printer` and `PrintJob` interfaces, an alternative approach is to build the API using opaque handles for printers and print jobs, similar to how the [`chrome.printing`](https://developer.chrome.com/docs/extensions/reference/printing/) API is designed.

```cs
dictionary PrinterHandle {
  // ...
};

dictionary PrintJobHandle {
  // ...
};

[Exposed=Window, SecureContext]
interface Printing {
  Promise<sequence<PrinterHandle>> getPrinters();

  Promise<PrinterAttributes> getPrinterAttributes(PrinterHandle);
  Promise<PrintJobHandle> printJob(
    PrinterHandle,
    DocumentDescription,
    optional JobTemplateAttributes = {});
  void cancelJob(PrintJobHandle);

  // event handlers, etc
};
```

Both approaches have their own pros and cons, which have been debated in various situations. However, we ultimately chose to proceed with an API design based on separate interfaces for printers and print jobs, as we believe this approach offers greater clarity and ease of use. By using this approach, the API consumer can clearly understand which methods and properties are associated with printers and which are associated with print jobs and organize the application logic accordingly.

### Weak Typing
The most popular NodeJS implementation of IPP ([ipp](https://www.npmjs.com/package/ipp) module) relies on weak typing for printer and job attributes. In other words, the proposed interface could be simplified to (note the `object` instead of `PrinterAttributes`/`JobTemplateAttributes`):
```cs
interface Printer {
  readonly attribute PrinterId id;

  // Corresponds to printer-name from RFC8011 §5.4.4
  // https://www.rfc-editor.org/rfc/rfc8011.html#section-5.4.4
  readonly attribute DOMString name;

  readonly attribute PrinterState state;
  // Plain event
  attribute EventHandler onstatechanged;

  Promise<object> getPrinterAttributes(
    optional sequence<OptionalPrinterAttribute> optionalAttributes = []);

  Promise<PrintJob> printJob(
    DOMString title,
    DocumentDescription document,
    optional object options = {});
};
```

While this approach can provide flexibility and ease of use in some cases, it can also lead to errors and unexpected behavior, especially in large and complex codebases. For this reason we decided to rely on the built-in WebIDL type checking, hence providing an additional layer of safety and reliability for the API.
* The generated code ensures that only correct types of data are passed between javascript and the API implementation, reducing the overhead of error-prone manual type checking.
* Enum/dictionary bindings improve language support.
* Erroneous calls reject early with a `TypeError` and offer developer-friendly error descriptions.

### Printer.cancel() as a promise
TBD

###
