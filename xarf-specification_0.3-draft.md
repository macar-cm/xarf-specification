# X-ARF Specification *v0.3-draft02*

This is a short specification for X-ARF (Extended Abuse Reporting Format). 
If you are interested in joining us and/or also start reporting in X-ARF or you think that you might have some valuable ideas, feel that there's still things missing etc., let us know. We are happy to get as much feedback as possible. 

## The idea of X-ARF

X-ARF is a format to report different types of network abuse incidents to network owners and other involved parties. It is designed as a pure container format capable of reporting different kinds of abuse.

Reports are composed of two or more parts encapsulated using MIME. The first part is a human-readable description of the incident. The second contains structured data intended for machine parsing. The third and subsequent parts contain any relevant evidence.

Like [MARF](http://www.rfc-editor.org/rfc/rfc5965.txt), the information in the second part of an X-ARF report is represented in [YAML](http://en.wikipedia.org/wiki/YAML). Each report type must define a JSON Schema [draft02](http://tools.ietf.org/html/draft-zyp-json-schema-02) (please notice!) for the YAML data so that a report can be validated. This is the _report type schema_. Along with the schema, a report type will usually need a document which describes the semantics of the fields described in the schema, specifies the evidence provided in the third and subsequent MIME parts and which are optional or mandatory, provides examples, explains the intent of each field, etc. This is the _report type guide_.

To make sure everybody can understand all available report types, this community has been established. This community discusses e.g. new report types and additional information for existing report types. Most members of the community are also active users of X-ARF for reporting and abuse handling. Publically available schemes are versioned and published on [x-arf.org](http://www.x-arf.org/schemata.html) and in a github [repository](https://github.com/abusix/xarf-schemata). That way every parser is able to validate against the latest X-ARF scheme and thus is able to handle it.
  
Please notice that this document is still in development. If you want to start reporting in X-ARF, feel free to join the X-ARF [mailinglist](http://lists.x-arf.org/cgi-bin/mailman/listinfo/) as well.

## Transport

This specification does not cover transport of X-ARF. X-ARF can easily be transported over SMTP, HTTP, or other protocols capable of file transfer. 

A transport description should specify if and how multiple X-ARF reports can be bundled together.

## Security

Earlier versions defined a "secure" report, which was simply an X-ARF report encapsulated in an S/MIME or PGP/MIME message. What's the goal of "security"? Is it encryption during transport of the message? Is it authentication of the report sender? Is it to detect tampering with the report?

But in general, security should be provided by the transport, not by X-ARF itself.

## Identities in X-ARF

Identities in X-ARF should be expressed as URIs, for maximum flexibility. These URIs must express the _identity_ of the user or entity (e.g. a service) which produced the report. Individual report types must define which kinds of URIs are acceptable and may place restrictions on them.

For example,

* `mailto` (RFC6068) - in the fraud schema
* `sip`/`sips` (RFC3261) - in the RCS report schema
* `tel` (RFC3966) - in the RCS report schema

These restrictions are placed on each scheme:

### `mailto`

`mailto` URIs MUST contain only the "mailto:" and `to` elements of the `mailtoURI` rule defined in RFC 6068. Headers MUST NOT be included.

### `sip`/`sips`

`sip`/`sips` URIs MUST be address-of-record URIs for the user or entity which is making the report. These URIs SHOULD NOT include parameters (other than `user=phone` for `sip` URIs constructed from `tel` URIs) and MUST NOT include header fields.

`sip`/`sips` URIs derived from `tel` URIs MUST NOT include visual separators when placing the `tel` telephone-subscriber portion into the `sip`/`sips` userinfo part.

### `tel`

`tel` URIs MUST be global numbers prefixed with "+" as described in RFC 3966 section 5.1.4. The telephone-subscriber portion MUST NOT contain visual-separator characters. They SHOULD NOT contain parameters.

## Overview of an X-ARF Report

One X-ARF report corresponds to an incident of abuse. In some cases, an "incident" may include many individual events - for example, an attack attempting to brute-force a login may include many thousands of connections and login attempts. A report type guide should provide guidelines for whether and how to collect events into one report.

An X-ARF transport protocol should define headers so that software can recognize X-ARF reports, along with other appropriate information. For example, an SMTP transport should specify use of the Auto-Submitted header.

An X-ARF report is composed of multiple parts, encapsulated using MIME. Note that the MIME header fields report will usually be included in the transport headers, i.e. when transported over SMTP, the Content-Type header field will be in the email headers. A "bare" X-ARF report should include the MIME header fields necessary to parse the report before the body parts.

An X-ARF report has content-type multipart/mixed. It must contain at least two MIME parts:

### 1st MIME part
* Human readable text which contains at least basic information about the reported incident
* <code>Content-Type: text/plain</code>
* <code>charset=utf-8</code>
* MUST be present

### 2nd MIME part
* The data is a JSON object (a list of key/value pairs) represented in [YAML](http://www.yaml.org) notation. See [Common keys and their semantics](#common-keys-and-their-semantics) for more information.
* Receivers must be able to validate against the provided JSON scheme (see [X-ARF tools](http://www.x-arf.org/tools.html)).
* Machine- and human-readable
* <code>Content-Type: text/plain</code>
* <code>charset=utf-8</code>
* <code>name="report.txt"</code>
* MUST be present

It may contain additonal MIME parts:

### 3rd and additional MIME parts
* Of any MIME types (which are specified in a referenced scheme)
* Contains e.g. evidence which may be anonymized
* Of any content
* Optional or mandatory as defined in the corresponding report type schema and guide

## Common YAML keys and their semantics

##### Schema-URL: 

All schemata must require one instance of this key.

This key is used to identify the report type. It contains the URI to the JSON-schema that describes the content of the report. The schema must be published under and the URI must point to the www.x-arf.org domain.

* Example: <code>Schema-URL: http://www.x-arf.org/schema/abuse_login-attack_0.1.2.json</code>

##### Version:

All schemata must require one instance of this key.

This field contains the version of this specification.  

*The current version number is 0.3*

* Example: <code>Version: 0.3</code> 

##### Reported-From:

All schemata should require one instance of this key.

This field contains the sender's identity, expressed as a URI.

* Examples:
** `Reported-From: mailto:reporter@example.com`
** `Reported-From: tel:+15551234`
** `Reported-Fro: sip:user@example.com`

##### User-Agent:

All schemata should require one instance of this key.

Name and version of the generating software (see RFC1945 and RFC2068) 

* Example: <code>User-Agent: xarf-reporter/1.0 (X-ARF Reporting Toolset v1.0)</code>

##### Report-ID:

All schemata must require one instance of this key.

The intent is that if a report is sent to multiple recipients - e.g. several different contact addresses at one ISP - they can be deduplicated. Thus the Report-ID for a report containing (one) specific incident(s) should be unique across time and space.

The Report-ID SHOULD have a reasonable domain part. In some contexts, such as mobile messaging, there will not be an obvious domain name to use. In that case, the home network domain name (3GPP TS 23.003) SHOULD be used as the domain part.

* It is recommended to use a combination of a (compact) UUID and your domain: <code>[any unique ID]@yourdomain.tld</code> 
* Example: `Report-ID: e9e1fd60c855012f30ab60c5470a79c4@yourdomain.tld`

##### Incident-Date:

All schemata should recommend no more than one instance of this key.

This field contains the date of the incident itself, as best can be determined. The date should be in RFC 3339 format - if the x-arf schema specifies the date with "format":"date-time". Due to compatibility reasons the date may be written in the RFC2822 format, no matter if "format":"date-time" is used or not in a x-arf schema description. Therefore, parser implementations should check which of the both formats is used. 

* Example (RFC2822 format): <code>Incident-Date: Mon, 05 Aug 2012 16:19:15 -0000</code> 
* Example (RFC3339 format): <code>Incident-Date: 2012-04-12T23:20:50.52Z</code> 

##### Report-Date:

All schemata should recommend no more than one instance of this key.

This field contains the date when the report was generated. The date should be in RFC 3339 format - if the x-arf schema specifies the date with "format":"date-time". Due to compatibility reasons the date may be written in the RFC2822 format, no matter if "format":"date-time" is used or not in a x-arf schema description. Therefore, parser implementations should check which of the both formats is used. 

* Example (RFC2822 format): <code>Report-Date: Mon, 05 Aug 2012 16:19:15 -0000</code> 
* Example (RFC3339 format): <code>Report-Date: 2012-04-12T23:20:50.52Z</code> 

##### Source:

All schemata should recommend no more than one instance of this key.

Contains the source of abusive behavior. This SHOULD be a URI - e.g. HTTP URL, sip, tel, or mailto URI, etc. If the source is a host or IP address, the source SHOULD be expressed as a string similar to a URI, with a "scheme" followed by an address. In this form the "scheme" MUST be "host", "ipv4", or "ipv6", followed by a colon, followed by the IP address or host name, optionally followed by a colon and port number.

* Examples: 
  * `Source: ipv4:192.168.1.134`
  * `Source: ipv4:192.168.1.134:9876`
  * `Source: ipv6:2001:898:2000:c:213:206:89:190`
  * `Source: https://www.domain.tld/folder/file.xxx`
  * `Source: host:domain.tld`
  * `Source: mailto:localpart@domain.tld`
  * `Source: tel:+15559876`

##### TLP:

Schemata may recommend no more than one instance of this key.

This field may be used to indicate the sensitivity of the information in the report.
Please see the [definition of Information Sharing Traffic Light Protocol](http://www.trusted-introducer.org/ISTLPv11.pdf).   
 
* Example: <code>TLP: amber</code>   

##### Attachment:

Those report types that allow additional evidence must require one instance of this key. This key is an array which contains content-type descriptors for each attachment in the message, in the same order as the attachments.

A report type may specify that some types are optional and some are required.

For example, a report type guide may say:

Reports of incidents of bad puns MUST include the pun itself an an attachment of type text/plain, and MAY include a photograph of the audience reaction as an image/jpeg or image/png.

A conforming report would include this key:

`Attachment: [ text/plain ] `

Or:

`Attachment: [ text/plain, image/jpeg ] `


##### Category:

Schemata may recommend no more than one instance of this key.

If a report type can encompass different classes, categories, or kinds of incident, this should be expressed in the Category field.

For example, a schema for spam messaging may list categories of "newsletter", "phishing", or "419 scam".

##### Occurrences:

Schemata may recommend no more than one instance of this key.

In many cases, an incident will occur some number of times over some period - for example, an attacker may attempt a brute-force login attack against some server. `Occurrences` expresses how many substantially-identical instances of abuse occurred.

* Example: <code>Occurrences: 6</code> 

##### Incident-Lifetime:

Schemata may recommend no more than one instance of this key.

In many cases, an incident will occur some number of times over some period - for example, an attacker may attempt a brute-force login attack against some server. `Incident-Lifetime` expresses the time period over which the abuse occurred. It expresses an amount of time in seconds, hours, or days.

* Examples:
    * `Incident-Lifetime: 500 seconds`
    * `Incident-Lifetime: 3 days`

## Changelog of the X-ARF specification
### Changes from v0.3 draft01 to draft02
* Updated "idea" description of X-ARF.
* Define a "plain" report as MIME headers followed by the report body parts
* Reordered sections
* Transport is now separate from format; an X-ARF transport should also define security and bulk format.
* Update URIs as identities to be explanatory rather than normative; the schema will specify details of URIs
* Updated overview to remove header sections - those should be described in a transport document
* Removed appendix A, since secure and bulk should be defined in a transport document
* Updated appendix B to be a description of common YAML keys and semantics; report type schemata and guides should implement these
* Renamed Appendix C to A, and updated it to illustrate only the "plain" X-ARF report
### Changes from v0.3 draft00 to draft01
* Updated to use URIs instead of email addresses for reporter identity
* Relaxed requirement for domain-name in Report-ID from "must" to "SHOULD", for enviroments where there is no obvious domain.
* Allow multiple "evidence" MIME parts
* Update "Source" field definition to use something like a URI, and discard "Source-Type" field.
* Split Date into Reported-Date and Incident-Date
* Add Incident-Lifetime field
### Changes from v0.1 to v0.2
* As of version v0.2 of this specification encrypted, signed, and bulk abuse reports are introduced.
* The identifier of `X-ARF: YES` changed to `X-XARF: PLAIN`, `X-XARF: SECURE`, and `X-XARF: BULK` to avert conflicts with old importers. Also, the additional `X` in `X-XARF` is a necessity to fulfill the requirements of a future RFC.
* Version v0.2 is still backward compatible with v0.1; therefore all v0.2 importers can handle X-ARF v0.1 generated messages with `X-ARF: YES` as identifier. The only alteration for v0.1 X-ARF reports is the `X-XARF: PLAIN` identifier. The v0.1 header `X-ARF: YES` is deprecated and should not be used in newer X-ARF generators. [Chapter 1](#chapter-1-plain-x-arf-messages) is in accordance with specification version 0.1.
* Recommendation for signing X-ARF messages with DKIM unless otherwise digitally secured.


## Appendix A: Visualisation of X-ARF messages

### `X-XARF: PLAIN`
A default plain message with a 3rd MIME part will look like:

![X-XARF PLAIN example](https://raw.github.com/abusix/xarf-specification/master/examples/v0.2/xarf-specification_0.2.x-xarf-plain.png)

