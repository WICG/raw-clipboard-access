### Questions from https://www.w3.org/TR/security-privacy-questionnaire/


## 2. Questions to Consider

Generally, note that secure contexts, focused frames, and user permissions are required to access this feature.


### 2.1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

Raw Clipboard Access will expose the contents of the platform clipboard, without sanitization. This exposure is necessary to allow web and native applications’ clipboard contents to interoperate. This would allow, for example, non-standardized document and image formats to be used by web applications. The existing status quo is that web and native applications transfer such information via downloading/uploading, or via serializing the information into the clipboard’s text/plain field. This will provide a new, more streamlined way for such web and native applications to transfer information between one another.


### 2.2. Is this specification exposing the minimum amount of information necessary to power the feature?

Yes, this information is necessary for web and native applications to effectively transfer non-standardized types between one another.


### 2.3. How does this specification deal with personal information or personally-identifiable information or information derived thereof?

PII may be transferred on the clipboard, as Raw Clipboard Access exposes all types, unsanitized. A web application may be able to identify secret information about a user, which a native application had put onto the clipboard unencrypted, with the assumption that no other program would read this information. 

While this is a concern, this information is considered less dangerous than clipboard text, which is already commonly used to transfer passwords, social security numbers, and other more-dangerous PII. Therefore, this API does not negatively deviate from the existing clipboard implementation in protecting the users’ PII.

In addition, just as with the existing Clipboard API, this transfer is only possible with user permission. This will require a permission prompt. As with the existing Async Clipboard API, Raw Clipboard Access will only be available in secure contexts and focused frames.


### 2.4. How does this specification deal with sensitive information?

Concerns and mitigations regarding sensitive information are the same as with PII.


### 2.5. Does this specification introduce new state for an origin that persists across browsing sessions?

Raw Clipboard Access exposes more potential state for an origin persisting across browsing sessions, in that new clipboard types will be exposed, and that such types may be unsanitized. This should be mitigated with the use of a permission, and availability only in secure contexts.


### 2.6. What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?

Clipboard data and formats available on the platform clipboard are exposed behind a permission prompt, making active fingerprinting possible. From the clipboard data, it could be possible to infer a potential range of operating system versions given the types available after a write, for platforms which convert clipboard types implicitly ([example](https://docs.microsoft.com/en-us/windows/win32/dataxchg/clipboard-formats#synthesized-clipboard-formats)), and given that the platform’s mappings change. Platform clipboard implicit mappings do not change often, so this platform information is only speculatively possible to fingerprint. This should not be much information, but in combination with navigator.userAgent, could help fingerprint a user with more precision. That said, this is significantly less powerful or useful than navigator.userAgent, and requires a permission to trigger, so should be relatively safe. This is consistent across origins.


### 2.7. Does this specification allow an origin access to sensors on a user’s device

No.


### 2.8. What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.

No data is exposed to other origins without (1) a copy/write in one origin, (2) a user clicking into another origin to make it the focused frame, (3) a user allowing a permission prompt to paste/read, and (4) a paste/read in the other origin. See 2.1 for data exposed in this instance.


### 2.9. Does this specification enable new script execution/loading mechanisms?

It is possible for new scripts to be loaded, as this clipboard information could be any arbitrary data. This is no different from the status quo, where users may paste code (example: `sudo rm -rf \`) into a terminal or editor and execute it.

A potentially enabled script execution surface is the instance in which a user may paste encoded data into native applications which have vulnerable decoders. For example, a malicious image might be pasted into an old application with an insecure decoder, and on paste, the image will be decoded, and remote code execution may occur. A permission prompt is in place for both reads and writes to ensure that the user takes care when allowing raw clipboard access.

User agents should make the risks of granting the permission clear to users.


### 2.10. Does this specification allow an origin to access other devices?

No.


### 2.11. Does this specification allow an origin some measure of control over a user agent’s native UI?

No.


### 2.12. What temporary identifiers might this this specification create or expose to the web?

None.


### 2.13. How does this specification distinguish between behavior in first-party and third-party contexts?

As with the Async Clipboard API, Raw Clipboard Access is only available in secure contexts and the focused frame.


### 2.14. How does this specification work in the context of a user agent’s Private \ Browsing or "incognito" mode?

There is no variation for private or incognito modes.


### 2.15. Does this specification have a "Security Considerations" and "Privacy Considerations" section?

Yes. Security Considerations are [here](https://www.w3.org/TR/clipboard-apis/#security), and Privacy Considerations are [here](https://www.w3.org/TR/clipboard-apis/#privacy).


### 2.16. Does this specification allow downgrading default security characteristics?

No.


### 2.17. What should this questionnaire have asked?

#### 2.17.1. How might this specification compromise a user's system? ([issue](https://github.com/dway123/raw-clipboard-access/issues/3))

Exposing raw clipboard content to the open web platform poses serious security issues, in that this introduces a large surface of unsanitized content, previously not available to the open web.

There are known security vulnerabilities in native applications' decoders. These decoders are often run when contents are pasted into such applications, and these vulnerabilities may allow for remote code execution with all the priviledges granted to the native application.

Example: A malicious web application may write an image with a payload designed to take advantage of insecure decoders. As raw clipboard access does not specify sanitization of clipboard contents prior to write, this will be written exactly as delivered by the web application. This malicious image might then be pasted into an application with an insecure decoder. When the user pastes into this application, the image will be decoded, and remote code execution outside of the sandboxed browser may occur. 

A permission prompt is in place for writes to ensure that the user takes care when allowing raw clipboard access. User agents should make the risks of granting the permission clear to users.

## 3. Threat Models


### 3.1. Passive Network Attackers

No new information visible to a passive network attacker is exposed by the Raw Clipboard Access API.


### 3.2. Active Network Attackers

An active network attacker would require active user intervention via a permission prompt, in order to gain read access to the clipboard’s contents. This is no different from the existing Clipboard API.


### 3.3. Same-Origin Policy Violations

This data isn’t associated with any origin, and is associated with the system instead. At minimum, a click into the other origin’s frame, secure context, and a previously granted permission is required to have an other origin gain access to this data. This is no different from the existing Clipboard API.


### 3.4. Third-Party Tracking

This should not be affected by third-party tracking, unless a custom clipboard type were to be used to attempt to track user behavior. However, is already more possible and powerful to do through cookies. See 3.5 Legitimate Misuse for more information.


### 3.5. Legitimate Misuse

It is possible that a Web or Native application secretly sends information the user did not intend to expose, through a custom clipboard type that is not expected to be used. While it is already possible to transmit data a user may not intend to transmit via text/plain and other standardized types, or via steganography in an image, this would provide another avenue to hide information on a platform clipboard.
