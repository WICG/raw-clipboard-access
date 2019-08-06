# **Async Clipboard: Raw Clipboard Access**

**Author:**
*   [huangdarwin@chromium.org](mailto:huangdarwin@chromium.org) 

## Introduction

Web Applications would like to copy and paste arbitrary data types, for which native applications may have vulnerable encoders/decoders. Raw Clipboard Access is a low-level API solution to allow for copying and pasting of data of these arbitrary Clipboard types. This unlocks compatibility with native apps, by allowing use of their non-web-standard formats such as TIFF (a large image type not standardized on the web), .docx (a proprietary document format), SVG (an image format without a secure Chrome encoder/decoder), etc. This also unlocks faster clipboard interactions, as re-encoding will not be required, allows for more custom types, and allows the user agent to avoid potential issues with re-encoding, like dropping metadata or changing image contents. The existing Async Clipboard API’s re-encoding is still encouraged for use cases requiring only generic types, and easier to use as custom encoders/decoders would not be necessary, but raw clipboard access allows web applications with more specific or sophisticated clipboard support needs to meet those needs.

## Goals

*   Allow copy/paste between web and native apps.
    *   These types will not be sanitized by the browser.
    *   They must be placed on the operating system clipboard, to allow for communication between web and native apps.
    *   Potential use cases:
         *   Copy/Paste of SVG images between [Figma](https://crbug.com/150835#c73) and Photoshop/GIMP.
         *   Copy/Paste of documents/spreadsheets/presentations between Microsoft Office 365 / Google Docs and Microsoft Office / Open Office.
*   Provide more fine-grained control over the clipboard, by allowing the web to:
    *   Skip decoding on write.
    *   Skip encoding on read.
    *   Control order of writing items to the clipboard.
*   Build on existing Async Clipboard API, by leveraging existing:
    *   structure, like ClipboardItem.
    *   async nature, permissions model, and secure-context/active frame requirement of the API.
*   Preserve security / privacy, as unsanitized data entering the operating system clipboard may be dangerous.

## Non-goals

*   Modify design of original Async Clipboard API, where not relevant to raw clipboard access.
*   Anything else not related to Async Clipboard API

## Existing Async Clipboard API write

The existing [Async Clipboard API](https://w3c.github.io/clipboard-apis/#async-clipboard-api) already provides for reading or writing multiple sanitized items from or to the clipboard. 

Existing Async Clipboard write call:
```javascript
const image = await fetch('myImage.png');
const text = new Blob(['this is an image'], {type: 'text/plain'});
const clipboard_item = new ClipboardItem({'text/plain' : text, 'image/png', image});
await navigator.clipboard.write([clipboard_item]);
```

Existing Async Clipboard read call:
```javascript
const clipboardItems = await navigator.clipboard.read();
const clipboardItem = clipboardItems[0];
const text = await clipboardItem.getType('text/plain');
const image = await clipboardItem.getType('image/png');
```

## Raw Clipboard Access Write

Clipboard representations are added just as in the Async Clipboard API, but the ordering of representations now informs the order in which each representation is written. Additionally, different operating systems have different names and representations for each clipboard format, so it’s recommended that the web application check `navigator.clipboard.platform` before interacting with the raw clipboard, and encode or decode information appropriately.


Example of this new write:
```javascript
// Basic raw clipboard write example.
const imageResponse = await fetch('myImage.png');
const image = await imageResponse.blob();
const text = new Blob(['this is an image'], {type: 'text/plain'});
// The developer should ensure that items are appropriately encoded/decoded 
// for the platform the web app is running on.
if(navigator.clipboard.platform !== 'Windows') { return; }
const clipboard_item = new ClipboardItem({
  'text/plain' : text, /* This first item in the dict will be written first. */
  'image/png' : image   /* This second in the dict will be written second. */
}, 
{raw : true} // This is an optional argument, which defaults to false. 
             // The entire write / ClipboardItem must be either re-encoded or raw.
);
await navigator.clipboard.write([clipboard_item]);

// More complete, hypothetical implementation example.
const imageResponse = await fetch('myImage.png');
const image = await imageResponse.blob();
let clipboard_item;

if(navigator.clipboard.platform === 'Windows') {
  const windows_image = await encode_jpeg_windows(image); // contains windows-only headers and carriage returns.
  const windows_image_xr = await encode_jpeg_xr_windows(image); // new higher-fidelity image format.
  clipboard_item = new ClipboardItem(
    {'image/jpg-xr' : windows_image_xr, 'image/jpg' : windows_image},
    {raw : true}
  );
}
else if(navigator.clipboard.platform === 'MacOS') {
  // macos_image_xr encoder not available in this hypothetical example (maybe legal reasons).
  const macos_image = await encode_tiff_macos(image); // contains macos-only headers.
  clipboard_item = new ClipboardItem({'image/tiff' : macos_image}, {raw : true});
}
else {
  // No x11 support in this hypothetical example .
  // (maybe this application was ported from an application with no available x11 encoder).
  clipboard_item = new ClipboardItem({'image/png' : image}, {raw : false});
}
await navigator.clipboard.write([clipboard_item]);
```

## Raw Clipboard Access Read

`Navigator.clipboard.read` gains an optional `raw` parameter as well, to inform whether the ClipboardItem returned should contain raw or encoded data and types. Once again, `navigator.clipboard.platform` can be used to determine the platform, and inform the format, in which the data may be encoded.


Example of this new read:
```javascript
// retrieves all items directly if raw:true, or all encoded items if raw:false 
// (raw defaults to false).
// raw set here, and also sets raw property in ClipboardItems.
const clipboardItems = await navigator.clipboard.read({raw:true});
const clipboardItem = clipboardItems[0];

const jpg = await clipboardItem.getType('image/jpg');
let image;
if (navigator.clipboard.platform === 'Windows'){
  image = convertForWindows(jpg);
}
else if(navigator.clipboard.platform === 'MacOS') {
  image = convertForMac(jpg);
}

if(image) // If image was successfully converted, draw it.
  draw_jpg(image);
```

## navigator.clipboard.platform

A new `navigator.clipboard.platform` API can determine the clipboard implementation currently in use, and will feature values like `“Windows”`, `“MacOS”`, `“ChromeOS”`, `“Android”`, `“X11 Linux”`, and `“iOS”`.

The existing `navigator.platform` with regular expression matching could notably also fulfill this clipboard implementation detection, but the feature is known to be potentially [bloated and confusing](http://stackoverflow.com/q/19877924), and would not result in a 1-1 ([or close to 1-1](https://stackoverflow.com/a/19883965/7548103)) mapping between `navigator.platform` and required encoding, so it was proposed to use a new `navigator.clipboard.platform`. Then again, adding yet another method to find a platform may be unnecessary, especially considering that this is a low-level API, and will likely be used in conjunction with Javascript libraries, which may perform this parsing.

Example of this new platform API:
```javascript
if(navigator.clipboard.platform === 'Windows') {
  // Only enter this statement on Windows.
}
else if(navigator.clipboard.platform === 'MacOS'){
  // Only enter this statement on MacOS.
}
...
```

## Design

### Allowed types per origin

As Windows (through [`RegisterClipboardFormatA`](https://docs.microsoft.com/en-us/windows/desktop/api/winuser/nf-winuser-registerclipboardformata)) and X11 (through Atoms) both limit the amount of clipboard formats they may register, browsers must take care to avoid registering too many types. Windows has the smallest limit, at about 2<sup>14</sup> [unique clipboard types](https://docs.microsoft.com/en-us/windows/desktop/api/winuser/nf-winuser-registerclipboardformata) supported. Therefore, I propose a limit of 32 (2<sup>5</sup>) types per origin, and a limit of 4096 (2<sup>12</sup>) total types registered through the Chrome Clipboard (including raw clipboard access). 32 was the number chosen, because this should be enough for a hypothetical document editor, like Office 365, Open Office, or Docs, to write all types that they currently support (Slides, Sheets, Documents, Drawings, Formulae, etc on all platforms, plus other generic types). This should allow for 128 (2<sup>7</sup>) unique origins to register the maximum amount of clipboard types available, or more origins to register less than the maximum amount of clipboard types available. There are ~1800 unique MIME types [registered](https://www.iana.org/assignments/media-types/media-types.xhtml), so while this wouldn’t come close to allowing each web application to register every available registered MIME type, it should be more than enough for most use cases. If future use suggests that more MIME types should be exposed to each origin, it should be much easier to expand the amount of exposed MIME types without breaking existing use cases, than it would be to reduce them if there were too many exposed.


### Clipboard Type Naming

Different operating systems have different limitations on the naming of their clipboard types. For example, Windows Clipboard Formats are [type insensitive](https://docs.microsoft.com/en-us/windows/desktop/api/Winuser/nf-winuser-registerclipboardformata#remarks), and MacOS uses [Uniform Type Identifiers](https://en.wikipedia.org/wiki/Uniform_Type_Identifier) to describe clipboard types, which means that these names may only contain “ASCII characters A-Z, a-z, 0-9, hyphen ("-"), and period ("."), and Unicode characters above U+007F” ([source](https://en.wikipedia.org/wiki/Uniform_Type_Identifier)). Raw Clipboard Access will pass these types directly to the system clipboard, without checking appropriate type naming per platform, so the web application needs to be careful not to use inappropriate names. If multiple names resolve to the same platform clipboard name, then the last one will overwrite the first, or multiple reads of the same data may occur.

In [Windows](https://cs.chromium.org/chromium/src/ui/base/clipboard/clipboard_format_type_win.cc?l=78), [X11](https://cs.chromium.org/chromium/src/ui/gfx/x/x11_atom_cache.cc?l=266), [macOS](https://cs.chromium.org/chromium/src/ui/base/clipboard/clipboard_format_type_mac.mm?l=140), and [Android](https://cs.chromium.org/chromium/src/ui/base/clipboard/clipboard_format_type_android.cc?l=102), registering arbitrary types can be accomplished by passing the type in as a string. To allow for compatibility with other native applications, the browser can pass the type straight through, and expect the native application to read/write the same type. _Type lengths will be limited to 1024 (2<sup>10</sup>) characters_, and must not parse to an integer, to avoid potential attempts to attack via sending very large type strings, and to attempt to ensure some sort of descriptive naming, though this could certainly be expanded with time. 

### Alternative: Consistent MIME types without re-encoding

Chrome could alternatively pass a Clipboard type through to the operating system, with a requirement that the Clipboard type be a standardized MIME type, with a consistent representation across platforms. As no operating system should have built in Clipboard types in MIME format, native applications will have to send their data using a MIME type format to access this on the web, or vice versa. This has the advantage of not requiring a `navigator.clipboard.platform` API. It also requires native applications to “opt in” to using these types, as they will have to potentially add support for these clipboard formats. Similarly, as is done for many similar features, web applications will also have to “opt in” to using these previously unavailable MIME clipboard types. This should provide for an organic growth of the web platform’s supported MIME types; as [Blob](https://developer.mozilla.org/en-US/docs/Web/API/Blob)s don’t mind which MIME type they contain (for most types), neither will the clipboard. It should also bypass fingerprinting concerns, as the type would be represented in the same manner- both in name and binary representation- across operating systems. However, this approach ultimately wouldn’t improve the web platform as much, because web applications would have to wait for native applications to update and add support for MIME types on the clipboard. As some native applications have very slow update cycles, and existing native applications are often not updated, this would cause for adoption of this API to be unnecessarily slow and expensive. Additionally, some native applications may have no desire to integrate with web applications, and may therefore choose to omit clipboard MIME type support for this purpose.


### Alternative: Use a new array type instead of ClipboardItem

Although ES2015 [preserves](https://www.stefanjudis.com/today-i-learned/property-order-is-predictable-in-javascript-objects-since-es2015/#2-strings-that-are-no-integers-) object property insertion order for string properties, this is not true of strings that parse to integers. Therefore, continuing to use `ClipboardItem` with the bracket notation necessitates excluding any clipboard type that parses directly to an integer.

Arrays clearly specify ordering regardless of the data in the array, and could more explicitly express the ordering of Clipboard Representations written to the clipboard. An array of pairs (2-object arrays) could very explicitly specify the (1) order of clipboard items, of (2) string key types and value blob data. Raw Clipboard Access could therefore use a new `RawClipboardItem` type, with array pairs, to specify ordering of writing to the system clipboard, instead of using `ClipboardItem`. This also means that integer keys could be supported.

This was decided against to simplify and minimize the API surface, by avoiding creating yet another Clipboard-specific object, which would only be used for Raw Clipboard. Additionally, no clipboard types supported by Windows and MacOS should parse directly to an integer anyways. Therefore, the potential use case is extremely limited, and would dramatically increase the complexity with minimal gain.

## Stakeholder Feedback / Opposition

Stakeholders like Figma are in support of, and have publicly [requested](https://crbug.com/150835#c73) this feature, to attain compatibility with native applications and avoid decoding. 

No public signals from implementers.

## References & acknowledgements

[Unless you have a specific reason not to, these should be in alphabetical order.]

Many thanks for the valuable feedback and advice from:

*   [Design Doc](https://tinyurl.com/raw-clipboard-access-design) reviewers:
    *   [dcheng@chromium.org](mailto:dcheng@chromium.org)
    *   [garykac@chromium.org](mailto:garykac@chromium.org) 
    *   [jsbell@chromium.org](mailto:jsbell@chromium.org)
    *   [palmer@chromium.org](mailto:palmer@chromium.org)
    *   [pwnall@chromium.org](mailto:pwnall@chromium.org)
    *   [reillyg@chromium.org](mailto:reillyg@chromium.org)
*   Reference Documents:
    *   [W3C explainer template](https://github.com/w3ctag/w3ctag.github.io/blob/master/explainers.md#explainer-template)
    *   [Design Document](https://tinyurl.com/raw-clipboard-access-design)
