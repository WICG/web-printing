# Web Printing Explainer

## Introduction

The `navigator.printing` Web API is a proposed standard for enabling printer-related functionality in web applications. This API provides a set of JavaScript methods that allow developers to query local printers, submit print jobs, and manage print job options and status directly from web applications. To represent these concepts, it relies on the attribute names and semantics from the Internet Printing Protocol (IPP) specifications.

Printing is a common and important task in many domains, including document management, label printing, receipt printing, and more. While web applications can generate printable content such as PDFs, images, and text documents and send them to local printers via `window.print()`, customizing those documents jobs beyond what can be accomplished declaratively via CSS or specifying additional parameters like the number of copies or collation is currently not possible; the `navigator.printing` API aims to provide a standardized and seamless way for web applications to directly interact with printers, enabling developers to build printer-related features within their web applications.

The explainer outlines the motivating use cases for the API, its key features, and how it can benefit web developers and users alike.

## Current State of Printing
While `window.print()` offers basic printing functionality through the browser's default print dialog, it is a relatively simple API: developers' capabilities are restricted to selecting a file for printing, applying limited customization via CSS and triggering the print dialog. Some of the inconveniences that arise as a result are discussed in this section; it's important to note that some of them can be significantly improved even without adopting a different printing model.

### Streamlining Workflows
First and foremost, repetitive printing tasks cannot be properly streamlined: there's no way to preconfigure a simple task like "print 10 double-sided copies of a document on letter paper" for daily use. These settings could be provided as optional parameters to `window.print()` and enhance the printing workflow.

### Knowing Your Printers
To enable accurate customization of settings, developers need to have knowledge about the capabilities of the printer. The proposed API provides a convenient method for accessing this information, which enhances the printing experience by allowing developers to retrieve relevant details about what the printer can and cannot do and apply this knowledge appropriately. A few examples:
* If the printer supports high-resolution printing, the application can automatically select the appropriate print resolution, resulting in sharper and more detailed output.
* Access to supported media sizes empowers the application to generate print-ready files side by adjusting page layouts, margins, or scaling, effectively ensuring that the content fits properly on the printed page without any cropping or distortion.

### Observing the Printing Pipeline & Improving Error Handling
When using the `window.print()` function for web printing, the developers do not have any visibility into the printing process at all, whereas the users are limited to browser notifications and obscure system UI surfaces that are not always easy to discover. This can lead to a frustrating experience for users, for instance, if they encounter errors but do not receive any guidance on how to resolve them, or if the print job gets stuck in the queue for long.

While these problems could be somewhat softened by improving the existing notification system and making it easier for users to understand the progress of print jobs, allowing the applications to monitor them directly could also be beneficial for the users in a number of ways.
* With more control over error handling, applications can wrap up raw printing errors into more elaborate descriptions (what went wrong and why?), suggest useful tips for troubleshooting common errors (potentially even going as far as providing links to vendor websites or user manuals for specific printers) or even prompt to print anew after the problem has been fixed.
* Granting the application visibility into the progress of print jobs would allow the apps to present this information in a more graphical & convenient manner.

## Main Motivating Use Case: Remote Printing
The incremental improvements to the existing printing stack discussed in the previous sections do not address the crucial case of remote printing which has specific requirements that do not fit into the general web document-based model.

* Remote Rendering

  In many cases the document being printed is rendered entirely on the remote system; the implication is that the local system receives a finalized media (such as a PDF document) which must be printed as-is rather than a raw version of the document in HTML that can be tailored by the means of the browser to the final print media available locally using `@media print`. With the existing printing stack, the best-effort approach for a remote app would be to ask the user to provide a generic description of their printers' capabilities, but it would lead to a significantly worse user experience than letting the remote application do what it usually does with the appropriate knowledge of the capabilities of local printers -- this is where the proposed API steps in.

* Customizing the Remote Print Job

  The remote print job might come with additional parameters such as copies and duplex settings when configured by the user when printing via a virtual printer which should be honored by the local system with a user confirmation -- there's no way to forward this information to the local printing system other than prompt the user to select the necessary settings in the print dialog box. One option would be to ehnance `window.print()` by providing optional parameters to pre-populate values in the local print dialog as suggested earlier in the explainer; an alternative solution would be to declare a virtual printer that doesn't support any settings and force all of such decisions to be made through the local print dialog which might be a bit more concise. Either approach results in a tedious two-stage printing process involving way more user interactions that necessary.

The proposed API methods unlock proper printer forwarding by allowing the remote client to access essential information about printers on the near side, eliminating the need for tedious manual configurations and excessive user interactions and significantly improving the remote printing experience.

## Further Use Cases
The remote printing case is the core driving force of this proposal; however, we acknowledge that there might be other use cases which also require this architectural shift -- all ideas and suggestions are welcome!

## Key Features
The API primarily focuses on listing local printers and their capabilities and sending print jobs to them.

Three new interfaces are introduced as part of the API:
- The `Printing` interface is a singleton that can be accessed as `navigator.printing`.
  - `Promise<sequence<Printers>> navigator.printing.getPrinters()` enumerates local printers and returns a list of `Printer` objects.
- The `Printer` interface provides a way to interact with printers: retrieve their properties and capabilities, initiate print jobs, and monitor printer state changes.
  - `Promise<PrinterAttributes> fetchAttributes()` implements the `Get-Printer-Attributes` IPP operation and returns a promise that is settled once the operation completes. To reduce the number of printer connections, we offer an additional `PrinterAttributes cachedAttributes()` method within the `Printer` class that is initially populated from the OS cache and then gets updated every time `fetchAttributes()` is invoked.
  - `Promise<PrintJob> printJob(...)` implements the `Print-Job` IPP operation and allows developers to customize the print job via a subset of supported job template attributes specified in section [5.2 of RFC8011](https://www.rfc-editor.org/rfc/rfc8011#section-5.2).
- The `PrintJob` interface provides a way to interact with print jobs: monitor their state changes and cancel on demand.
  - `void cancel()` can be invoked to attempt to cancel a print job.
  - `PrintJobAttributes attributes()` (and dictionary fields like `job-state`, `job-state-message` and `job-state-reasons`) and `attribute EventHandler onjobstatechange` allow developers to track the job state and wait for its completion.

## IPP Mapping

Attribute names/values in the proposed API are modelled with respect to the existing IPP names as specified in [Internet Printing Protocol (IPP) Registrations](https://www.iana.org/assignments/ipp-registrations/ipp-registrations.xhtml). This choice is primarily dictated by the remote printing use case, since the remote system is more likely to be familiar with the IPP abstractions rather than higher-level CSS abstractions, which in turn reduces the number of intermediate translation steps by the components involved.

The following table specifies WebIDL type mapping for selected types listed in section [5.1 of RFC8011](https://www.rfc-editor.org/rfc/rfc8011#section-5.1).

| IPP Type | WebIDL Type |
| :-------------: | :-------------: |
| `text` | `DOMString` |
| `name` | `DOMString` |
| `keyword` | `enum` |
| `enum` | `enum` |
| `keyword` \| `name` | `DOMString` |
| `uri` | `DOMString` |
| `boolean` | `boolean` |
| `integer` | `unsigned long` |
| `1setOfX` | `sequence<X>` |
| `rangeOfInteger` | `dictionary { unsigned long from, unsigned long to };` |
| `octetString` | `DOMString` |
| `resolution` | `dictionary {`<br>`unsigned long cross-feed-direction-resolution,`<br>`unsigned long feed-direction-resolution,`<br> `PrintingResolutionUnits units`<br>`};` |
| `collection` | `dictionary { ... }` |

Attribute names retain their hyphen-case notation as `dictionary` members.

Names for `enum` classes are inferred as follows:
* Take the attribute name as base term;
* Translate kebab-case into CamelCase;
* If the term doesn't already start with `Print` or `Printing`, prepend `Printing`;
* Declare an enum with the derived term as name.

A few examples:
* `media-source` is backed by `enum PrintingMediaSource`;
* `print-color-mode` is backed by `enum PrintColorMode`.

Enum values are supposed to be translated from IPP to WebIDL without any additional changes.
However, the implementation is allowed to only support a selected subset of values for any given enum.

## WebIDL

```cs
[Exposed=Window, SecureContext]
interface PrintJob {
  PrintJobAttributes attributes();

  attribute EventHandler onjobstatehange;

  void cancel();
};

[Exposed=Window, SecureContext]
interface Printer {
  PrinterAttributes cachedAttributes();

  Promise<PrinterAttributes> fetchAttributes();

  Promise<PrintJob> printJob(
    DOMString title,
    PrintingDocumentDescription document,
    optional PrintJobTemplateAttributes options = {});
};

[Exposed=Window, SecureContext]
interface Printing {
  Promise<sequence<Printer>> getPrinters();
};

[Exposed=Window, SecureContext]
partial interface Navigator {
  [SameObject] readonly attribute Printing printing;
};
```

<details>
<summary>IDL Dictionaries & Enums (click to expand)</summary>

### Dictionaries
<details>
<summary>Supporting Dictionaries (click to expand)</summary>

```cs
dictionary PrintingMediaSize {
  unsigned long x-dimension;
  unsigned long y-dimension;
};

dictionary PrintingMediaCol {
  PrintingMediaSize media-size;
  DOMString media-size-name;
  PrintingMediaSource media-source;
};

dictionary RangeOfInteger {
  unsigned long from;
  unsigned long to;
};

dictionary PrintingResolution {
  unsigned long cross-feed-direction-resolution;
  unsigned long feed-direction-resolution;
  PrintingResolutionUnits units;
};

dictionary PrintingDocumentDescription {
  required Blob data;
  PrintingMimeMediaType document-format;
};
```

</details>

```cs
dictionary PrintJobTemplateAttributes {
  unsigned long copies;
  PrintingMedia media;
  PrintingMediaCol media-col;
  PrintingMultipleDocumentHandling multiple-document-handling;
  PrintingOrientation orientation-requested;
  sequence<PrintingRange> page-ranges;
  PrintingResolution printer-resolution;
  PrintColorMode print-color-mode;
  PrintQuality print-quality;
  PrintingSides sides;
};

dictionary PrintJobAttributes {
  DOMString job-id;
  DOMString job-name;

  DOMString printer-id;
  DOMString printer-name;

  PrintJobState job-state;
  DOMString job-state-message;
  sequence<PrintJobStateReason> job-state-reasons;
};

dictionary PrinterAttributes {
  unsigned short copies-default;
  PrintingRange copies-supported;

  PrintingMimeMediaType document-format-default;
  sequence<PrintingMimeMediaType> document-format-supported;

  sequence<PrintingJobCreationAttribute> job-creation-attributes-supported;

  PrintingMedia media-default;
  sequence<PrintingMedia> media-ready;
  sequence<PrintingMedia> media-supported;

  sequence<PrintingMediaCol> media-col-ready;
  sequence<PrintingMediaColAttribute> media-col-supported;

  PrintingMultipleDocumentHandling multiple-document-handling-default;
  sequence<PrintingMultipleDocumentHandling> multiple-document-handling-supported;

  PrintingOrientation orientation-requested-default;
  sequence<PrintingOrientation> orientation-requested-supported;

  boolean page-ranges-supported;

  DOMString printer-id;
  DOMString printer-name;
  DOMString printer-make-and-model;

  boolean printer-is-default;

  PrinterState printer-state;
  DOMString printer-state-message;
  sequence<PrinterStateReason> printer-state-reasons;

  PrintingResolution printer-resolution-default;
  sequence<PrintingResolution> printer-resolution-supported;

  PrintColorMode print-color-mode-default;
  sequence<PrintColorMode> print-color-mode-supported;

  PrintQuality print-quality-default;
  sequence<PrintQuality> print-quality-supported;

  PrintingSides sides-default;
  sequence<PrintingSides> sides-supported;
};
```

### Enums
<details>
<summary>Enums (click to expand)</summary>

```cs
enum PrintingMimeMediaType {
  "application/pdf",
};

enum PrintingMultipleDocumentHandling {
  "separate-documents-collated-copies",
  "separate-documents-uncollated-copies",
};

enum PrintColorMode {
  "monochrome",
  "color",
};

enum PrintingMediaColSupported {
  "media-size",
  "media-size-name",
  "media-source",
};

enum PrintingJobCreationAttributesSupported {
  "copies",
  "sides",
  "orientation-requested",
  "media",
  "print-quality",
  "printer-resolution",
  "output-bin",
  "media-col",
  "output-mode",
  "page-ranges",
  "multiple-document-handling",
  "print-color-mode",
};

enum PrintingMediaSource {
  "auto",
  // ...
};

enum PrintingPageOrientation {
  "3",
  "4",
  "5",
  "6",
};

enum PrintingResolutionUnits {
  "3",
  "4",
};

enum PrintingSides {
  "one-sided",
  "two-sided-long-edge",
  "two-sided-short-edge",
};

enum PrintJobState {
  "preparing",
  "pending",
  "processing",
  "canceled",
  "aborted",
  "completed",
};

enum PrinterState {
  // ...
};

typedef DOMString PrintingMedia;

```

</details>

</details>


## Usage Examples

### Listing Printers
```js
try {
  const printers = await navigator.printing.getPrinters();
  printers.forEach(printer => {
    printer.fetchAttributes().then(attributes => {
      console.log(
        `${attributes['printer-name']} has the following attributes: ${attributes}`);
    });
  });
} catch (err) {
  console.warn("Printing operation failed: " + err);
}
```

### Querying Printer State
```js
try {
  const printers = await navigator.printing.getPrinters();
  const printer = printers.find(
    printer => printer.cachedAttributes()['printer-name'] === 'Brother QL-820NWB');
  const attributes = await printer.updateAttributes();
  console.log(
    `${attributes['printer-name']}'s new state is ${attributes['printer-state']}!`);
} catch (err) {
  console.warn("Printing operation failed: " + err);
}
```

### Submitting a Print Job
```js
try {
  const printers = await navigator.printing.getPrinters();
  const printer = printers.find(
    printer => printer.cachedAttributes()['printer-name'] === 'Brother QL-820NWB');

  const printJob = await printer.printJob("Sample Print Job", {
    data: new Blob(...),
    'document-format': 'application/pdf',
  }, {
    copies: 2,
    media: 'iso_a4_210x297mm',
    'multiple-document-handling': 'separate-documents-collated-copies',
    'printer-resolution': {
      'cross-feed-direction-resolution': 300,
      'feed-direction-resolution': 400,
      units: '3' // dots per inch
    },
    sides: 'one-sided',
    'print-quality': '5' // best quality
    'page-ranges': [{from: 1, to: 5}, {from: 7, to: 10}],
  });

  const printJobComplete = new Promise((resolve, reject) => {
    printJob.onjobstatechange = () => {
      const jobState = printJob.attributes()['job-state'];
      if (IsErrorStatus(jobState)) {
        console.warn(`Job errored: ${jobState}`);
        reject(/**/);
        return;
      }
      if (jobState === PrintJobState.COMPLETED) {
        console.log("Job complete!");
        resolve(/**/);
        return;
      }
      console.log(`Job state changed to ${jobState}`);
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
  const printer = printers.find(
    printer => printer.cachedAttributes()['print-name'] === 'Brother QL-820NWB');

  const printJob = await printer.printJob(...);

  // This might take no effect if the job has already finished.
  printJob.cancel();

} catch (err) {
  console.warn("Printing operation failed: " + err);
}
```

## Security & Privacy Considerations

The scenarios described here refer to API access by websites in general and can be read as a justification for the recommendation of having it available only to [Isolated Web Apps](https://github.com/reillyeon/isolated-web-apps/blob/main/README.md). Note that despite this recommendation, this API is described independently from it.

### Fingerprinting
The detailed information about the available local printers provide an [active fingerprinting](https://www.w3.org/TR/fingerprinting-guidance/#dfn-active-fingerprinting) surface. This is possible via the combination of `navigator.printing.getPrinters()` + `Printer.fetchAttributes()` API calls and does not require user consent. Some of these printer values are (but not limited to):
* `printer-id`
* `printer-name`
* `printer-make-and-model`

It's worth noting that printers are usually default-configured in private spaces and hence don't provide too much insight; however, the situation is often different in corporate environments where printers are often customly tailored by admins; thus any deviation from standard settings for a particular model might potentially give away a corporate user. Moreover, the same logic applies to printer tiers: the presence of an expensive enterprise-scale printer (as opposed to a common customer model) allows the website to accurately determine a business user.

A more intricate fingerprinting approach is to constantly track `printer-state`/`printer-state-message` fields for a specific printer; by observing the patterns in printer state changes (i.e. how often it goes from `busy` to `idle` and vice versa) a macilious website could reveal sensitive information about the user's printing behavior. This information could be used to build a profile of the user or to track their activities across multiple sites.

#### Mitigation
A basic precaution is to introduce a new permissions-policy called `"printing"` to `navigator.printing.getPrinters()`  to facilitate protection against malicious third-party iframes.
A more sophisticated approach would include creating a permissions surface resembling those for existing Web APIs like `serial` and `hid` and prompting the user to select the printers that a particular origin is allowed to access with an option to revoke them on demand.

### Forging Printer Jobs
A malicious website could use the `printJob()` method to create and send fake printer jobs to the user's printer, causing it to waste paper and ink or potentially print malicious content; this could even potentially happen with a legitimate website due an inaccuracy in client-side printer handling.

#### Mitigation
Submitting print jobs will require an explicit consent from the user similar to the print dialog shown during `window.print()`.

### DoS-ing the System
A website might accidentally organize an unexpected DoS-attack by constantly calling `fetchAttributes()` and overloading the network as a result - this is particularly important in the enterprise setting with hundreds of printers.

#### Mitigation
Implementors should consider adding rate limiting to printer interactions.

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
    optional PrintJobTemplateAttributes = {});
  void cancelJob(PrintJobHandle);

  // event handlers, etc
};
```

Both approaches have their own pros and cons, which have been debated in various situations. However, a decision was made to adopt an API design that utilizes distinct interfaces for printers and print jobs thus providing better clarity and ease of use. By using this approach, the API consumer can clearly understand which methods and properties are associated with printers and which are associated with print jobs and organize the application logic accordingly.

### Weak Typing
The most popular NodeJS implementation of IPP ([ipp](https://www.npmjs.com/package/ipp) module) relies on weak typing for printer and job attributes. In other words, the proposed interface could be simplified to (note the `object` instead of `PrinterAttributes`/`PrintJobTemplateAttributes`):
```cs
interface Printer {
  readonly attribute object attributes;

  Promise<void> updateAttributes();

  Promise<PrintJob> printJob(
    DOMString title,
    DocumentDescription document,
    optional object options = {});
};
```

While this approach can provide flexibility and ease of use in some cases, it can also lead to errors and unexpected behavior, especially in large and complex codebases. For this reason a better option would be to rely on the built-in WebIDL type checking, hence providing an additional layer of safety and reliability for the API.
* The generated code ensures that only correct types of data are passed between javascript and the API implementation, reducing the overhead of error-prone manual type checking.
* Enum/dictionary bindings improve language support.
* Erroneous calls reject early with a `TypeError` and offer developer-friendly error descriptions.

### onprinterstatechange EventHandler for Printer
Similar to how the `PrintJob` objects provides a subscription for state change events, we could introduce an `onprinterstatechange` to allow tracking printer state changes, effectively utilizing the `Create-Printer-Subscription` IPP functionality. However, since the API is mostly centered around creating & tracking print jobs, we believe it would be sufficient for the developers to query the up-to-date printer state right before submitting a job; another alternative for the developer would be to set up regular calls to `updateAttributes()` via `setInterval()` for selected printers. However, this assumption is not set in stone and might change in the future depending on the developer feedback.

### `attributes` as interface member for Printer and PrintJob
It was originally planned to introduce `attributes` as an interface member to both `Printer` and `PrintJob` in a uniform fashion instead of wrapping them into getter functions; unfortunately, it's not possible to make a dictionary
an interface member in WebIDL.
