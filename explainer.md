# **Async Clipboard: Raw Clipboard Access**

**Author:**
*   [huangdarwin@chromium.org](mailto:huangdarwin@chromium.org) 

## Introduction

Powerful web applications would like to exchange data with native applications via the OS clipboard (copy-paste). The existing Web Platform has a high-level API that supports the most popular standardized data types (text, image, rich text) across all platforms. However, this API does not scale to the long tail of specialized formats. In particular, non-web-standard formats like TIFF (a large image format), and proprietary formats like .docx (a document format), are not supported by the current Web Platform. 

Raw Clipboard Access aims to provide a low-level API solution to this problem, by implementing copying and pasting of data with any arbitrary Clipboard type, without encoding and decoding.

This could be used by:
* Online editors like Google Docs or Microsoft Office 365, copy/paste OpenOffice or Microsoft Word documents/spreadsheets/presentations (proprietary formats).
* [Figma](https://crbug.com/150835#c73) or Photopea, to copy/paste PhotoShop/GIMP, GIFs, or RAW images, or already-supported formats with not-supported metadata (long tail of metadata).
* Web Apps supporting “niche” types, like LaTeX, .ogg, etc (long tail of formats).

The existing Async Clipboard API’s re-encoding is still encouraged for use cases requiring only generic types, and easier to use as custom encoders/decoders would not be necessary, but raw clipboard access allows web applications with more specific or sophisticated clipboard support needs to meet those needs.

## Goals

*   Allow copy/paste between web and native apps.
    *   These types will not be sanitized by the browser.
    *   These types must be placed on the operating system clipboard, to allow for communication between web and native apps.
    *   It must be feasible to adopt such support in a reasonable time frame.
*   Provide fine-grained control over the clipboard, by allowing the web to:
    *   Skip decoding on write by user agent.
    *   Skip encoding on read by user agent.
    *   Control order of writing formats to the clipboard.
    *   Create custom clipboard formats.
*   Build on existing Async Clipboard API, by leveraging existing:
    *   structure, like ClipboardItem.
    *   async nature, permissions model, and secure-context/active frame requirement of the API.
*   Preserve security / privacy, as unsanitized data interacting with the operating system clipboard may be dangerous.

## Non-goals

*   Modify design of original Async Clipboard API, where not relevant to raw clipboard access. This includes modifying the permission model of the Async Clipboard API, such as its non-requirement of user gesture or persistence. While these may be valid concerns, they are out of scope of this explainer.
*   Anything else not related to Async Clipboard API.

## Existing Async Clipboard API read and write

The existing [Async Clipboard API](https://w3c.github.io/clipboard-apis/#async-clipboard-api) already provides for reading or writing multiple sanitized items from or to the clipboard. 

Existing Async Clipboard write call:
```javascript
const image = await fetch('myImage.png');
const text = new Blob(['this is an image'], {type: 'text/plain'});
const clipboard_item = new ClipboardItem({'text/plain': text, 'image/png': image});
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

Clipboard representations are added just as in the Async Clipboard API, but the ordering of representations now informs the order in which each representation is written. It is recommended to write platform-independent code if possible, but different operating systems may have different names and representations for each clipboard format. Therefore, if a web application is reading or writing such platform-dependent formats, it's recommended that they check `navigator.clipboard.platform` before interacting with the raw clipboard, and encode or decode information appropriately.


Example of this new write:
```javascript
// Basic raw clipboard write example.
const imageResponse = await fetch('myImage.png');
const image = await imageResponse.blob();
const proprietaryFormat = await proprietaryEncode(image);
const text = new Blob(['this is an image'], {type: 'text/plain'});

const clipboard_item = new ClipboardItem({
  'image/png': image, /* This first item in the dict will be written first. */
  'text/plain': text, /* This second in the dict will be written second. */
  'my/format': proprietaryFormat /* This format may not be supported without Raw Clipboard Access */
}, 
{raw: true} // This is an optional argument, which defaults to false. 
            // The entire write / ClipboardItem must be either re-encoded or raw.
);
await navigator.clipboard.write([clipboard_item]);

// More complete, hypothetical implementation example.
const imageResponse = await fetch('myImage.png');
const image = await imageResponse.blob();
let clipboard_item;

// The developer should ensure that items are appropriately encoded/decoded 
// for the platform the web app is running on.
if(navigator.clipboard.platform === 'Windows') {
  // contains windows-only headers and carriage returns.
  const windows_image = await encode_jpeg_windows(image);
  // new higher-fidelity image format.
  const windows_image_xr = await encode_jpeg_xr_windows(image);
  clipboard_item = new ClipboardItem(
    {'image/jpg-xr': windows_image_xr, 'image/jpg': windows_image},
    {raw: true}
  );
} else if (navigator.clipboard.platform === 'MacOS') {
  // macos_image_xr encoder not available in this hypothetical example (maybe legal reasons).
  const macos_image = await encode_tiff_macos(image); // contains macos-only headers.
  clipboard_item = new ClipboardItem({'image/tiff': macos_image}, {raw: true});
} else {
  // In this hypothetical example, X11, Android, iOS, ChromeOS, and other platforms 
  // default to only write 'image/png', perhaps due to there being no specialized 
  // compatability needs that led to the use of raw clipboard access in 
  // Windows and MacOS.
  clipboard_item = new ClipboardItem({'image/png': image}, {raw: false});
}
await navigator.clipboard.write([clipboard_item]);
```

## Raw Clipboard Access Read

`Navigator.clipboard.read` gains an optional `raw` parameter as well, to inform whether the ClipboardItem returned should contain raw or encoded data and types. Once again, `navigator.clipboard.platform` can be used to determine the platform, and inform the format, in which the data may be encoded, but it is recommended to avoid platform-dependent code if possible.

Example of this new read:
```javascript
// retrieves all items directly if raw:true, or all encoded items if raw:false 
// (raw defaults to false).
// raw set here, and also sets raw property in ClipboardItems.
const clipboardItems = await navigator.clipboard.read({raw: true});
const clipboardItem = clipboardItems[0];
if(clipboardItem.types.length != 1 || clipboardItem.types[0] != 'image/jpg') {
  return;
}

const jpg = await clipboardItem.getType('image/jpg');
let image;
if (navigator.clipboard.platform === 'Windows') {
  image = convertForWindows(jpg);
} else if (navigator.clipboard.platform === 'MacOS') {
  image = convertForMac(jpg);
} else {
  // This jpg has a platform-independent decoder, but the app may wish to 
  // preserve certain hypothetical non-standardized, platform-dependent
  // data on Windows and MacOS for compatability with certain applications.
  image = generalConvert(jpg);
}

if(image) // If image was successfully converted, draw it.
  draw_jpg(image);
```

## navigator.clipboard.platform

A new `navigator.clipboard.platform` API can determine the clipboard implementation currently in use, and will feature values like `“Windows”`, `“MacOS”`, `“ChromeOS”`, `“Android”`, and `“X11 Linux”`.

The existing `navigator.platform` with regular expression matching could notably also fulfill this clipboard implementation detection, but the feature is known to be potentially [bloated and confusing](http://stackoverflow.com/q/19877924), and would not result in a 1-1 ([or close to 1-1](https://stackoverflow.com/a/19883965/7548103)) mapping between `navigator.platform` and required encoding, so it was proposed to use a new `navigator.clipboard.platform`. Then again, adding yet another method to find a platform may be unnecessary, especially considering that this is a low-level API, and will likely be used in conjunction with Javascript libraries, which may perform this parsing.

Example of this new platform API:
```javascript
if (navigator.clipboard.platform === 'Windows') {
  // Only enter this statement on Windows.
} else if (navigator.clipboard.platform === 'MacOS') {
  // Only enter this statement on MacOS.
}
...
```

Please note that `navigator.clipboard.platform` should only be used if it is absolutely necessary for the web application to exercise platform-dependent code, as it will significantly complicate web application code, and may reduce future compatability.

## Design

### Allowed types per origin

As Windows (through [`RegisterClipboardFormatA`](https://docs.microsoft.com/en-us/windows/desktop/api/winuser/nf-winuser-registerclipboardformata)) and X11 (through Atoms) both limit the amount of clipboard formats they may register, browsers must take care to avoid registering too many types. Windows has the smallest limit, at about 2<sup>14</sup> [unique clipboard types](https://docs.microsoft.com/en-us/windows/desktop/api/winuser/nf-winuser-registerclipboardformata) supported. Therefore, I propose a limit of 32 (2<sup>5</sup>) types per origin, and a limit of 4096 (2<sup>12</sup>) total types registered through the user agent clipboard (including raw clipboard access). 32 was the number chosen, because this should be enough for a hypothetical document editor, like Office 365, Open Office, or Docs, to write all types that they currently support (Slides, Sheets, Documents, Drawings, Formulae, etc on all platforms, plus other generic types). This should allow for 128 (2<sup>7</sup>) unique origins to register the maximum amount of clipboard types available, or more origins to register less than the maximum amount of clipboard types available. There are ~1800 unique MIME types [registered](https://www.iana.org/assignments/media-types/media-types.xhtml), so while this wouldn’t come close to allowing each web application to register every available registered MIME type, it should be more than enough for most use cases. If future use suggests that more MIME types should be exposed to each origin, it should be much easier to expand the amount of exposed MIME types without breaking existing use cases, than it would be to reduce them if there were too many exposed.

### Clipboard Type Naming

Different operating systems have different limitations on the naming of their clipboard types. For example, Windows Clipboard Formats are [type insensitive](https://docs.microsoft.com/en-us/windows/desktop/api/Winuser/nf-winuser-registerclipboardformata#remarks), and MacOS uses [Uniform Type Identifiers](https://en.wikipedia.org/wiki/Uniform_Type_Identifier) to describe clipboard types, which means that these names may only contain “ASCII characters A-Z, a-z, 0-9, hyphen ("-"), and period ("."), and Unicode characters above U+007F” ([source](https://en.wikipedia.org/wiki/Uniform_Type_Identifier)). Raw Clipboard Access will pass these types directly to the system clipboard, without checking appropriate type naming per platform, so the web application needs to be careful not to use inappropriate names. If multiple names resolve to the same platform clipboard name, then the last one will overwrite the first, or multiple reads of the same data may occur.

In [Windows](https://cs.chromium.org/chromium/src/ui/base/clipboard/clipboard_format_type_win.cc?l=78), [X11](https://cs.chromium.org/chromium/src/ui/gfx/x/x11_atom_cache.cc?l=266), [macOS](https://cs.chromium.org/chromium/src/ui/base/clipboard/clipboard_format_type_mac.mm?l=140), and [Android](https://cs.chromium.org/chromium/src/ui/base/clipboard/clipboard_format_type_android.cc?l=102), registering arbitrary types can be accomplished by passing the type in as a string. To allow for compatibility with other native applications, the browser can pass the type straight through, and expect the native application to read/write the same type. _Type lengths will be limited to 1024 (2<sup>10</sup>) characters_, and must not parse to an integer, to avoid potential attempts to attack via sending very large type strings, and to attempt to ensure some sort of descriptive naming, though this could certainly be expanded with time. 

### Protections

As with the Async Clipboard API, which Raw Clipboard Access builds upon, Raw Clipboard Access will require secure context, active frame, and an active user permission prompt. In addition, user agents should take care to limit the amount of allowed types per origin, to avoid concerns with Windows and X11 format registration limits.

### Alternative: Consistent MIME types without re-encoding / Pickling
The user agent could alternatively pass a Clipboard type through to the operating system, with a requirement that the Clipboard type be a MIME type with a consistent representation across platforms. As no operating system should have built in Clipboard types in MIME format, native and web applications may converge to using these formats.

* (+) This requires native applications to explicitly “opt in” to using these types, as they will have to potentially add support for these clipboard formats. Therefore, legacy formats not intended for the web - leaking PII or containing decoder vulnerabilities - will not be exposed.
* (+) This does not requiring platform-dependent branching/code in the web, and therefore also avoids needing `navigator.clipboard.platform`. This also means reduced complexity in web application code.
* (-) This approach ultimately wouldn’t meet goals of interoperability between native and web application clipboard data, because web applications would have to wait for native applications to update and add support for MIME types on the clipboard. As some native applications have very slow update cycles (often >1 year), and existing native applications are often not updated, this would cause for adoption of this API to be unnecessarily slow and expensive.
* (-) This provides minimal new capability for the web platform, as opt-in information passing is already possible by transparently embedding such data in text/plain, or silently doing so through the unstandardized pickling format that Chrome and Webkit already expose (which is already done in the wild).

Pickling was not chosen as it does not meet the requirements desired for raw clipboard access. Namely, interoperability between native and web applications could not be assured within a reasonable time-frame through pickling.

### Alternative: Use a new array type instead of ClipboardItem

Although ES2015 [preserves](https://www.stefanjudis.com/today-i-learned/property-order-is-predictable-in-javascript-objects-since-es2015/#2-strings-that-are-no-integers-) object property insertion order for string properties, this is not true of strings that parse to integers. Therefore, continuing to use `ClipboardItem` with the bracket notation necessitates excluding any clipboard type that parses directly to an integer.

Arrays clearly specify ordering regardless of the data in the array, and could more explicitly express the ordering of Clipboard Representations written to the clipboard. An array of pairs (2-object arrays) could very explicitly specify the (1) order of clipboard items, of (2) string key types and value blob data. Raw Clipboard Access could therefore use a new `RawClipboardItem` type, with array pairs, to specify ordering of writing to the system clipboard, instead of using `ClipboardItem`. This also means that integer keys could be supported.

This was decided against to simplify and minimize the API surface, by avoiding creating yet another Clipboard-specific object, which would only be used for Raw Clipboard. Additionally, no clipboard types supported by Windows and MacOS should parse directly to an integer anyways. Therefore, the potential use case is extremely limited, and would dramatically increase the complexity with minimal gain.

### Alternative: Restrict raw clipboard access to "partner" native/web applications

After some discussion with Webkit, a proposed alternative was to allow only "partner" sites, for example native and web applications with the same source origin, to have raw clipboard access. This was decided against for this explainer as it would break the compatibility requirements for Raw Clipboard Access, but may be a viable reduced implementation for user agents concerned about the full described API that may satisfy some common use-cases.

### Minimal implementation for user agents
Feature detection should be possible by checking `ClipboardItem`'s prototype for the `raw` property, and may be helpful to detect whether to use Raw Clipboard Access is implemented, in environments where not all user agents have implemented. That said, Webkit has [suggested](https://github.com/dway123/raw-clipboard-access/issues/3#issuecomment-538627436) that a user agent that decides not to implement all of Raw Clipboard Access may also consider rejecting on any call with `{raw : true}`, and instead fall back and write the payload as `{raw : false}`.

## Permissions

The [Clipboard spec](https://w3c.github.io/clipboard-apis/#clipboard-permissions)'s `ClipboardPermissionDescriptor` will be extended to add an `allowWithoutSanitization` field. Like with `allowWithoutGesture`, `allowWithoutSanitization === true` will be stronger than `allowWithoutSanitization === false`.

## Stakeholder Feedback / Opposition

- Web application stakeholders like Figma, Photopea, and SketchUp are public partners and in public support for this feature, to help attain compatibility and parity with native applications. For example, here's a [public request from Figma](https://crbug.com/150835#c73).
- Microsoft is [interested](https://groups.google.com/a/chromium.org/d/msg/blink-dev/rkGWbui8B9A/tCP8zINcDgAJ) [in](https://github.com/mozilla/standards-positions/issues/206#issuecomment-541267486) this feature.
- WebKit is [not in support of](https://github.com/dway123/raw-clipboard-access/issues/3#issuecomment-521016571) this feature.
- Mozilla is [not in support of](https://github.com/mozilla/standards-positions/issues/206) this feature.

## References & acknowledgements

Many thanks for the valuable feedback and advice from:

*   [Design Doc](https://tinyurl.com/raw-clipboard-access-design) reviewers:
    *   [dcheng@chromium.org](mailto:dcheng@chromium.org)
    *   [garykac@chromium.org](mailto:garykac@chromium.org) 
    *   [jsbell@chromium.org](mailto:jsbell@chromium.org)
    *   [mek@chromium.org](mailto:mek@chromium.org)
    *   [palmer@chromium.org](mailto:palmer@chromium.org)
    *   [pwnall@chromium.org](mailto:pwnall@chromium.org)
    *   [reillyg@chromium.org](mailto:reillyg@chromium.org)
*   Reference Documents:
    *   [W3C explainer template](https://github.com/w3ctag/w3ctag.github.io/blob/master/explainers.md#explainer-template)
    *   [Design Document](https://tinyurl.com/raw-clipboard-access-design)
