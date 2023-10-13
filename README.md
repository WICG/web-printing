# Web Printing Explainer

## Introduction

The `navigator.printing` Web API is a proposed standard for enabling deeper integration with printer-related functionality in web applications. This API provides a set of JavaScript methods that allow developers to query local printers, submit print jobs, and manage print job options and status directly from web applications. To represent these concepts, it relies on the attribute names and semantics from the Internet Printing Protocol (IPP) specifications.

Printing is a common and important task in many domains, including document management, label printing, receipt printing, and more. While web applications can generate printable content such as PDFs, images, and text documents and send them to local printers via `window.print()`, customizing those documents jobs beyond what can be accomplished declaratively via CSS or specifying additional parameters like the number of copies or collation is currently not possible; the `navigator.printing` API aims to provide a standardized and seamless way for web applications to directly interact with printers, enabling developers to build printer-related features within their web applications.

The explainer outlines the motivating use cases for the API, its key features, and how it can benefit web developers and users alike.

## Current State of Printing
While `window.print()` offers basic printing functionality through the browser's default print dialog, it is a relatively simple API: developers' capabilities are restricted to selecting a file for printing, applying limited customization via CSS and triggering the print dialog. Some of the inconveniences that arise as a result are discussed in this section; it's important to note that some of them can be significantly improved even without adopting a different printing model.

### Streamlining Workflows
First and foremost, repetitive printing tasks cannot be properly streamlined: there's no way to preconfigure a simple task like "print 10 double-sided copies of a document on letter paper" for daily use. These settings could be provided as optional parameters to `window.print()` and enhance the printing workflow.

### Knowing Your Printers
To enable accurate customization of settings, developers need to have knowledge about the capabilities of the printer. The proposed API provides a method for accessing this information, which enhances the printing experience by allowing developers to retrieve relevant details about what the printer can and cannot do and apply this knowledge appropriately. A few examples:
* If the printer supports high-resolution printing, the application can automatically select the appropriate print resolution, resulting in sharper and more detailed output.
* Access to supported media sizes empowers the application to generate print-ready files on the server-side by adjusting page layouts, margins, or scaling, effectively ensuring that the content fits properly on the printed page without any cropping or distortion.

### Observing the Printing Pipeline & Improving Error Handling
When using the `window.print()` function for web printing, the developers do not have visibility into the status of the print job. For the most part this is unnecessary because the browser and operating system provide indications of print progress to the user. For applications where printing is a core component of their function however this can prevent the developer from offering a better user experience. For example, an application for printing shipping labels might warn the user that a label failed to print and offer to re-print the label before moving on to the next package, reducing errors.

## Main Motivating Use Case: Remote Printing
The incremental improvements to the existing printing stack discussed in the previous sections do not address the crucial case of remote printing in remote/virtual desktop systems which has specific requirements that do not fit into the general web document-based model.

### Concepts & Definitions
* Remote system (also remote side/far side) is the desktop the user is connected to via the remote/virtual desktop system; it's assumed to be situated in a different location and not have access to local printers.
* Local system (also local side/near side) is the actual user's device which acts as a thin client/proxy to the remote system; it's assumed to be able to access local printers on the network.

### Challenges
Remote desktop environments (e.g. Citrix / Chrome Remote Desktop / etc) often need to print documents in the user's local environment. These documents usually come from various third-party apps (think of Microsoft Office, for instance), and the printing mechanism looks approximately as follows:
* The remote desktop environment installs a virtual printer on the remote side that shows up in the list of available printers;
* When the user clicks on "Print" from within the app and selects this virtual printer, the current document gets rendered into a file format suitable for printing with respect to the selected settings and forwarded to the local side;
* The remote desktop environment on the local side proceeds to initiate a real print job using the received document.

This process only allows a fully rendered document (e.g. pdf) to be transferred from the remote environment to the local one that must be printed as-is rather than a raw version of the document in HTML that can be tailored by the means of the browser to the final print media available locally using `@media print`; the natural conclusion is that knowing the printer capabilities beforehand is crucial to ensure correctness of the rendering itself.

With the existing printing stack, the best-effort approach for a remote app to approximate the printer capabilities would be to ask the user to provide a generic description of their printer and take it from there, yet it would lead to a significantly worse user experience than letting the remote application do what it usually does with the appropriate knowledge of the capabilities of local printers -- this is where the proposed API steps in.

Another challenge is transferring the print settings that users select. In line with the reasoning above, additional parameters essential for layout such as paper or color support must be configured before rendering takes place; however, there's no way to make the local print system honor these settings in the existing print stack other than prompt the user to select the necessary settings in the print dialog box. An alternative would be to ehnance `window.print()` by providing optional parameters to pre-populate values in the local print dialog as suggested earlier in the explainer.

As for the resulting duplication of print dialogs (one on the remote side for selecting the settings and another on the local side for confirming the print job), one of the solutions explored is to grant specific origins permission to print silently to a particular printer (with a provided option for the user to revoke their consent); please refer to the [Permissions UX](#permissions-ux) section for more details.

All in all, the proposed API methods unlock proper printer forwarding by allowing the remote client to access essential information about printers on the local side, eliminating the need for tedious manual configurations and excessive user interactions and significantly improving the remote printing experience.

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
  - `PrintJobAttributes attributes()` (and dictionary fields like `jobState`, `jobStateMessage` and `jobStateReasons`) and `attribute EventHandler onjobstatechange` allow developers to track the job state and wait for its completion.

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
| `resolution` | `dictionary {`<br>`unsigned long crossFeedDirectionResolution,`<br>`unsigned long feedDirectionResolution,`<br> `PrintingResolutionUnits units`<br>`};` |
| `collection` | `dictionary { ... }` |

Attribute names in collections are mapped to `dictionary` members as follows:
* Take the attribute name as base term;
* Translate kebab-case into lower camelCase.

A few examples:
* `cross-feed-direction-resolution` becomes `crossFeedDirectionResolution` in WebIDL.
* `print-color-mode` becomes `printColorMode` in WebIDL.

Names for `enum` classes are inferred as follows:
* Take the attribute name as base term;
* Translate kebab-case into upper CamelCase;
* If the term doesn't already start with `Print` or `Printing`, prepend `Printing`;
* Declare an enum with the derived term as name.

A few examples:
* `media-source` is backed by `enum PrintingMediaSource`.
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
  unsigned long xDimension;
  unsigned long yDimension;
};

dictionary PrintingMediaCol {
  PrintingMediaSize mediaSize;
  DOMString mediaSizeName;
  PrintingMediaSource mediaSource;
};

dictionary RangeOfInteger {
  unsigned long from;
  unsigned long to;
};

dictionary PrintingResolution {
  unsigned long crossFeedDirectionResolution;
  unsigned long feedDirectionResolution;
  PrintingResolutionUnits units;
};

dictionary PrintingDocumentDescription {
  required Blob data;
  PrintingMimeMediaType documentFormat;
};
```

</details>

```cs
dictionary PrintJobTemplateAttributes {
  unsigned long copies;
  PrintingMedia media;
  PrintingMediaCol mediaCol;
  PrintingMultipleDocumentHandling multipleDocumentHandling;
  PrintingOrientation orientationRequested;
  sequence<PrintingRange> pageRanges;
  PrintingResolution printerResolution;
  PrintColorMode printColorMode;
  PrintQuality printQuality;
  PrintingSides sides;
};

dictionary PrintJobAttributes {
  DOMString jobId;
  DOMString jobName;

  DOMString printerId;
  DOMString printerName;

  PrintJobState jobState;
  DOMString jobStateMessage;
  sequence<PrintJobStateReason> jobStateReasons;
};

dictionary PrinterAttributes {
  unsigned short copiesDefault;
  PrintingRange copiesSupported;

  PrintingMimeMediaType documentFormatDefault;
  sequence<PrintingMimeMediaType> documentFormatSupported;

  sequence<PrintingJobCreationAttribute> jobCreationAttributesSupported;

  PrintingMedia mediaDefault;
  sequence<PrintingMedia> mediaReady;
  sequence<PrintingMedia> mediaSupported;

  sequence<PrintingMediaCol> mediaColReady;
  sequence<PrintingMediaColAttribute> mediaColSupported;

  PrintingMultipleDocumentHandling multipleDocumentHandlingDefault;
  sequence<PrintingMultipleDocumentHandling> multipleDocumentHandlingSupported;

  PrintingOrientation orientationRequestedDefault;
  sequence<PrintingOrientation> orientationRequestedSupported;

  boolean pageRangesSupported;

  DOMString printerId;
  DOMString printerName;
  DOMString printerMakeAndModel;

  boolean printerIsDefault;

  PrinterState printerState;
  DOMString printerStateMessage;
  sequence<PrinterStateReason> printerStateReasons;

  PrintingResolution printerResolutionDefault;
  sequence<PrintingResolution> printerResolutionSupported;

  PrintColorMode printColorModeDefault;
  sequence<PrintColorMode> printColorModeSupported;

  PrintQuality printQualityDefault;
  sequence<PrintQuality> printQualitySupported;

  PrintingSides sidesDefault;
  sequence<PrintingSides> sidesSupported;
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
  // ...
};

enum PrintQuality {
  "draft",
  "normal",
  "high",
}

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
  "portrait",
  "landscape",
  "reverse-landscape",
  "reverse-portrait",
};

enum PrintingResolutionUnits {
  "dots-per-inch",
  "dots-per-centimeter",
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

### Listing Printers & Basic Attributes
```js
try {
  const printers = await navigator.printing.getPrinters();
  for (const printer of printers) {
    const attributes = printer.cachedAttributes();
    console.log(
      `${attributes.printerName} has the following (basic) attributes: ${attributes}`);
  }
} catch (err) {
  console.warn("Printing operation failed: " + err);
}
```

### Listing Printers & Detailed Attributes
```js
try {
  const printers = await navigator.printing.getPrinters();
  const promises = printers.map(printer => printer.fetchAttributes());
  Promise.all(promises).then((values) => {
    for (const attributes of values) {
      console.log(
        `${attributes.printerName} has the following (detailed) attributes: ${attributes}`);
    }
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
    printer => printer.cachedAttributes().printerName === 'Brother QL-820NWB');
  const attributes = await printer.fetchAttributes();
  console.log(
    `${attributes.printerName}'s new state is ${attributes.printerState}!`);
} catch (err) {
  console.warn("Printing operation failed: " + err);
}
```

### Submitting a Print Job
```js
try {
  const printers = await navigator.printing.getPrinters();
  const printer = printers.find(
    printer => printer.cachedAttributes().printerName === 'Brother QL-820NWB');

  const printJob = await printer.printJob("Sample Print Job", {
    data: new Blob(...),
    documentFormat: 'application/pdf',
  }, {
    copies: 2,
    media: 'iso_a4_210x297mm',
    multipleDocumentHandling: 'separate-documents-collated-copies',
    printerResolution: {
      crossFeedDirectionResolution: 300,
      feedDirectionResolution: 400,
      units: 'dots-per-inch'
    },
    sides: 'one-sided',
    printQuality: 'high',
    pageRanges: [{from: 1, to: 5}, {from: 7, to: 10}],
  });

  const printJobComplete = new Promise((resolve, reject) => {
    printJob.onjobstatechange = () => {
      const jobState = printJob.attributes().jobState;
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
    printer => printer.cachedAttributes().printerName === 'Brother QL-820NWB');

  const printJob = await printer.printJob(...);

  // This might take no effect if the job has already finished.
  printJob.cancel();

} catch (err) {
  console.warn("Printing operation failed: " + err);
}
```

## Security & Privacy Considerations

### Fingerprinting
The detailed information about the available local printers provide an [active fingerprinting](https://www.w3.org/TR/fingerprinting-guidance/#dfn-active-fingerprinting) surface. This is possible via the combination of `navigator.printing.getPrinters()` + `Printer.fetchAttributes()` API calls and does not require user consent. Some of these printer values are (but not limited to):
* `printerId`
* `printerName`
* `printerMakeAndModel`

It's worth noting that printers are usually default-configured in private spaces and hence don't provide too much insight; however, the situation is often different in corporate environments where printers are often customly tailored by admins; thus any deviation from standard settings for a particular model might potentially give away a corporate user. Moreover, the same logic applies to printer tiers: the presence of an expensive enterprise-scale printer (as opposed to a common customer model) allows the website to accurately determine a business user.

A more intricate fingerprinting approach is to constantly track `printerState`/`printerStateMessage` fields for a specific printer; by observing the patterns in printer state changes (i.e. how often it goes from `busy` to `idle` and vice versa) a macilious website could reveal sensitive information about the user's printing behavior. This information could be used to build a profile of the user or to track their activities across multiple sites.

#### Mitigation
A basic precaution is to introduce a new permissions-policy called `"printing"` to `navigator.printing.getPrinters()`  to facilitate protection against malicious third-party iframes.
A more sophisticated approach would include creating a permissions surface resembling those for existing Web APIs like `serial` and `hid`: this is discussed in greater detail in the [Permissions UX](#permissions-ux) section.

### Forging Printer Jobs
A malicious website could use the `printJob()` method to create and send fake printer jobs to the user's printer, causing it to waste paper and ink or potentially print malicious content; this could even potentially happen with a legitimate website due an inaccuracy in client-side printer handling.

#### Mitigation
Submitting print jobs will require an explicit consent from the user similar to the print dialog shown during `window.print()`.

### DoS-ing the System
A website might accidentally organize an unexpected DoS-attack by constantly calling `fetchAttributes()` and overloading the network as a result - this is particularly important in the enterprise setting with hundreds of printers.

#### Mitigation
Implementors should consider adding rate limiting to printer interactions.

## Permissions UX
Due to the privacy concerns associated with [fingerprinting](#fingerprinting), we need a new way for users to control how their printers are tracked and accessed; this calls for a new Permissions UX for the Printing API.

### Existing Permissions Flow
The existing web flow doesn't allow web sites to access printers directly -- instead the only prompt that the user gets is shown directly during the printing stage, allowing them to select a printer and customize the options on a one-off basis.

### Suggested Options
There are two main flows to be considered given that the API is also offering direct access to printers in addition to the ability to submit print jobs.

* Flow A: Grant Access to all Printers
  1. Prompt the user to grant permission to access all local printer information
  2. Site constructs a print job based on the capabilities of a printer it chooses
  3. Prompt the user when the job is submitted so that they can confirm selected printer and job options.

Flow A is analogous to permissions for microphone and camera devices, in which a single grant provides the website with information about all connected hardware, allowing it to select which device to use. However, unlike microphones and cameras, there may be multiple printers connected to a user's device, which presents a greater surface area for fingerprinting in enterprise environments. With that in mind, a more intricate Flow B is put forward:


* Flow B: Grant Access on a per-Printer Basis
  1. Prompt the user to select local printer(s) to share with the site
  2. Site constructs a print job based on the capabilities of a printer it chooses (among the ones that were selected by the user)
  3. Prompt the user again when a job is submitted to confirm selected options.

Flow B is more attractive due to its privacy-preserving nature: similar to other device APIs it exposes the least information possible about the user’s device configuration. The per-job prompt from bullet 3 can be additionally equipped with an option to "Do not show this confirmation again" if they trust a particular web site similar to how the "Multiple Downloads" option is designed.

### Enterprise Considerations
In all flows, enterprise policy may automatically grant or deny the prompts. Additionally, Flow B should respect enterprise settings to expose information about all available printers for selected origins.

### VDI Considerations
In virtual desktop infrastructure (VDI) scenarios involving printing, users may be presented with two separate printing dialogs: one for the remote host and one for the local device. The user experience (UX) should therefore consider the following:
* Differentiating the two dialogs sufficiently to make it clear that the local dialog is solely a confirmation dialog, rather than a prompt to select the same settings again.
* Preventing potential confusion related to clicking "Do not show this confirmation again" and clarifying that this setting only applies to the local dialog.

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

  Promise<void> fetchAttributes();

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
Similar to how the `PrintJob` objects provides a subscription for state change events, we could introduce an `onprinterstatechange` to allow tracking printer state changes, effectively utilizing the `Create-Printer-Subscription` IPP functionality. However, since the API is mostly centered around creating & tracking print jobs, we believe it would be sufficient for the developers to query the up-to-date printer state right before submitting a job; another alternative for the developer would be to set up regular calls to `fetchAttributes()` via `setInterval()` for selected printers. However, this assumption is not set in stone and might change in the future depending on the developer feedback.

### `attributes` as interface member for Printer and PrintJob
It was originally planned to introduce `attributes` as an interface member to both `Printer` and `PrintJob` in a uniform fashion instead of wrapping them into getter functions; unfortunately, it's not possible to make a dictionary
an interface member in WebIDL.
