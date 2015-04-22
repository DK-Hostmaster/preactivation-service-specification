# DK Hostmaster Pre-activation Service Specification

2015/04/22
Revision: 2.0

# Table of Contents

<!-- MarkdownTOC depth=3 -->

- [Introduction](#introduction)
- [About this Document](#about-this-document)
	- [Document History](#document-history)
- [The .dk Registry in Brief](#the-dk-registry-in-brief)
- [Pre-activation Service](#pre-activation-service)
	- [Migration from version 1 to 2](#migration-from-version-1-to-2)
	- [Available Environments](#available-environments)
		- [Production Environment](#production-environment)
		- [Sandbox Environment](#sandbox-environment)
- [Implementation Requirements](#implementation-requirements)
	- [General parameters (checksum)](#general-parameters-checksum)
	- [registrar data section](#registrar-data-section)
	- [registrant data section](#registrant-data-section)
	- [domain data section](#domain-data-section)
- [Checksum Calculation](#checksum-calculation)
	- [Example Checksum Calculation](#example-checksum-calculation)
- [Request Flow](#request-flow)
	- [on_accept](#on_accept)
		- [General parameters (status)](#general-parameters-status)
		- [registrar data section](#registrar-data-section-1)
		- [registrant data section (if validated)](#registrant-data-section-if-validated)
		- [registrant data section (if not-validated)](#registrant-data-section-if-not-validated)
		- [domain data section](#domain-data-section-1)
	- [on_reject](#on_reject)
		- [General parameters (status)](#general-parameters-status-1)
		- [registrar data section](#registrar-data-section-2)
		- [registrant data section](#registrant-data-section-1)
		- [domain data section](#domain-data-section-2)
	- [on_edit](#on_edit)
		- [General parameters (status)](#general-parameters-status-2)
		- [registrar data section](#registrar-data-section-3)
	- [on_fail](#on_fail)
		- [General parameters (status)](#general-parameters-status-3)
		- [registrar data section](#registrar-data-section-4)
		- [registrant data section](#registrant-data-section-2)
		- [domain data section](#domain-data-section-3)
	- [on_error](#on_error)
		- [General parameters (status)](#general-parameters-status-4)
		- [registrar data section](#registrar-data-section-5)
		- [registrant data section](#registrant-data-section-3)
		- [domain data section](#domain-data-section-4)
- [Validation](#validation)
- [Implementation Limitations](#implementation-limitations)
	- [Locale](#locale)
	- [Amount of Domain Names](#amount-of-domain-names)
	- [Shared-secret Handling](#shared-secret-handling)
	- [Encryption](#encryption)
	- [Sandbox Environment](#sandbox-environment-1)
		- [Test Data](#test-data)
- [References](#references)
- [Resources](#resources)
	- [Demo Client](#demo-client)
	- [Mailing list](#mailing-list)
	- [Issue Reporting](#issue-reporting)
	- [Additional Information](#additional-information)
- [Data Sheet](#data-sheet)

<!-- /MarkdownTOC -->

# Introduction

This document describes and specifies the implementation offered by DK Hostmaster for interaction with the central registry for the ccTLD dk using our pre-activation service. It is primarily aimed at a technical audience, and the reader is required to have prior knowledge of domain name registration.

# About this Document

This specification describes version 2 (2.X.X) of the service implementation. Future releases will be reflected in updates to this specification, please see the document history section below.

Any future extensions and possible additions and changes to the implementation are not within the scope of this document and will not be discussed or mentioned throughout this document.

This document is owned and maintained by DK Hostmaster A/S and must not be distributed without this information.

All examples provided in the document are fabricated or changed from real data to demonstrate commands etc. any resemblence to actual data are coincidental.

## Document History

* 2.0 2015-01-09
  * Updated to reflect the version 2 of the service
  * Updated checksum calculation example, the behaviour of the `echo` command had to be taken into consideration

* 1.1 2014-06-16 
  * No longer in draft. Added mention of demo client on Github.

* 1.0 2014-02-28 
  * Initial revision

# The .dk Registry in Brief

DK Hostmaster is the registry for the ccTLD for Denmark (dk). The current model used in Denmark is based on a sole registry, with DK Hostmaster maintaining the central DNS registry.

# Pre-activation Service

The DK Hostmaster pre-activation Service is based on a SOA architecture. The implementation is regarded as a service offered to external parties offering pre-activation to customers.

The service requires the use of and possible development of client-side software and integration. This is beyond the scope of this specification as the API and other assets for assisting in this are the primary object of this document.

In addition to the assets, DK Hostmaster aims to assist client, users and developers of client software with integration towards DK Hostmaster and therefore provide facilities to ease this integration. This is primarily centered around a sandbox environment and related documentation.

## Migration from version 1 to 2

* `registrar.url.on_fail` introduced, called of all attempts to validate data are exhausted

* `registrar.pnumber` introduced, for validation of danish legal entities having a CVR number (`registrant.vatnumber`), can be used in conjunction with as `registrant.pnumber` or or be left out

* `registrar.url.on_accept` and `registrar.url.on_reject` now get a complete data set returned from the service, since validation and data alteration is handles by the pre-activation service

* `registrar.language` has been deprecated, instead the service offers two end-points as dedicated URLs indicating the language to be used, the same works for the sandbox environment:

  * https://preact.dk-hostmaster.dk/en
  * https://preact.dk-hostmaster.dk/da

  * https://preact-sandbox.dk-hostmaster.dk/en
  * https://preact-sandbox.dk-hostmaster.dk/da

* All of the return data has been extended due to validation taking place so the data submitted might have changed over the course of the validation and is hence returned in the final and accepted state. Please note that this is primarily relevant where validation is taking place.

* The requirements for the callback has been tightened, so the endpoints are only called using TLS 1.0. This is due to the transport of the sensitive personal data

## Available Environments

DK Hostmaster offers the following environments (see also the Data sheet):

| Environment | Role | Description |
| ----------- | ---- | ----------- |
| production  | production | This environment is the production environment |
| sandbox     | development | This environment is intended for client development towards the DK Hostmaster pre-activation service. Also [the demo client][preact-demo-client] uses this environment. |

### Production Environment

In order to use the production environment, the registrar has to provide DK Hostmaster with a shared-secret. This will be appointed a key id, which will have to be used on all subsequent requests toward the production environment as `registrar.keyid`. Please see the section on Checksum calculation below.

### Sandbox Environment

The sandbox environment offers only a known dummy registrar-id and related key-id (`registrar.keyid`), so all requests to the sandbox environment should use this parameter in all requests and be presented as the demonstration registar.

* `registrar.keyid` = 999888

# Implementation Requirements

This section outlines the overall requirements in regard to implementing an pre-activation integration to work with the DK Hostmaster pre-activation service.
The service is available via HTTPS to the end-customer, meaning for domain name registration the associated registrant. The registrar integrates with the service by directing a request to the the services address bearing designated query parameters.

A request should consist of the following:

* checksum
* registrar data section
* registrant data section
* domain data section 

The checksum is calculated on a per request basis and relies on request data and predefined values. The latter is simply to protect the registrar userid for being transported in the open via third-party, see more on the checksum calculation in the section below.

The registrar data section consists on a set of data specifying handlers for the request flow of the request, not meaning the HTTP request, but the complete request. 

Please see the below section on request flow.

## General parameters (checksum)

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| `checksum` | yes | See section on checksum calculation |

## registrar data section

Please note that all end-urls should support TLS 1.0.

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| `registrar.keyid` | yes | Registrar key id, identifying a key held with the registry for validation of the request. |
| `registrar.reference` | yes | Reference for unique identification of the request from the registrar, equivalent of the mail forms field: 1b. |
| `registrar.transactionid` | yes | Registrars transactionid |
| `registrar.url.on_error` | yes | URL for error handling (see Request flow) |
| `registrar.url.on_edit` | yes | URL for handling edit request by the user(see Request flow) |
| `registrar.url.on_accept` | yes | URL for accept handling (see Request flow) |
| `registrar.url.on_fail` | yes | URL for validation failure handling (see Request flow) | 
| `registrar.url.on_reject` | yes | URL for rejection of request (see Request flow) |

## registrant data section

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| `registrant.userid` | yes (A) | Existing user-id, which can be associated with an active user with the registry. This user-id should point to the potential registrant. Equivalent of the mails forms field 4. |
| `registrant.type` | yes (B) | User type, should be either one of: C (company), P (Public Organisation), A (Association) or I (Individual). Equivalent of the mail forms field 4a. |
| `registrant.name` | yes (B) | Company, organization, association or person name, in reference to the above field. Equivalent of mail form field 4b. |
| `registrant.vatnumber` | no (B)* | VAT number, equivalent of mail form 4c. Mandatory for type C (company) and P (public organisation), can be provided for A (association) if the specified association has a VAT number. |
| `registrant.pnumber` | no (B)* | P-number. Mandatory for type C (company) and P (public organisation), can be provided for A (association) if the specified association has a p-number. |
| `registrant.address.street1` | yes (B) | Equivalent of mail form field 4f. |
| `registrant.address.street2` | no (B) | Equivalent of mail form field 4g. |
| `registrant.address.street3` | no (B) | Equivalent of mail form field 4h. |
| `registrant.address.zipcode` | yes (B) | Equivalent of mail form field 4i. |
| `registrant.address.city` | yes (B) | Equivalent of mail form field 4j. |
| `registrant.address.countryregionid` | yes (B) | Two-letter country code based on ISO 3166 Alpha 2. Equivalent of mail form field 4fk. (See references below). |
| `registrant.email` | yes (B) | Equivalent of mail form field 4l. |
| `registrant.phone` | yes (B) | Equivalent of mail form field 4m. |
| `registrant.telefax` | no (B) | Equivalent of mail form field 4n. |

## domain data section

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| `domain.N.name` | yes | Valid Danish domain name. N indicates a number between 1 and 10 since we only allow up to 10 domain names in the same request. |

# Checksum Calculation

The checksum calculation is based on the following parameters taken from the request and the data stored with DK Hostmaster.

* shared secret
* registrar userid
* registrars transaction id (`registrar.transactionid`)
* 1-10 domain names (`domain.N.name`)

These should be concatenated to a single string separated by `;` (semicolon)

## Example Checksum Calculation

Example with one domain name.

Strings:

```
shared secret: dkhm-sandbox-test-secret
registrar userid: REG-999999
registrar.transactionid: 1024
domain.1.name: æøå.dk
```

Concatenated string:

```
nolongersecret;reg-999999;1024;æøå.dk
```

```bash
$ echo -n 'dkhm-sandbox-test-secret;REG-999999;1024;æøå.dk' | shasum -a 256 
f74c49ad3133266bedaebc121a8f82ac81216414783c48d967d61dac15ac0fff
```

# Request Flow

The following diagram depicts the integration towards the service.

![request flow diagram][preact-request-flow-diagram]

The call-backs to the registrar are handled by the:

* `registrar.on_edit` - called if the user requests edition of the presented data
* `registrar.on_accept` - called when DK Hostmaster terms and conditions are accepted
* `registrar.on_reject` - called if the DK Hostmaster terms and conditions are not accepted
* `registration.on_error` - called in case of an non-critical exception where the error can be handled on the registrar side. 
* `registration.on_fail` - called in case of an validation attempts are exhausted.

Critical exceptions are presented locally. Examples of critical issues are:

  * Service issues
  * Possible security violations

The user is presented with the following page on the redirect:

![preactivation service screendump][preact-screendump]

The edit icon and links will direct back to the registrar for a page where the user can edit the presented data. The ‘I accept’ and ‘I decline’ also direct back to the registrar indicating the action taken by the end-user. When the page is initially called and if the request contains invalid data, under the conditions that they can be validated, will redirect to an error page with the registrar indicating the field not validating.

The data provided to the different handlers are described in detail below. The different handlers are used in different scenarios, there is however no restriction on where the URLs point and they can all 
point to the same end-point if this is requested.

## on_accept

This URL is called if the user decides to accept the request, please note that the returned data have been validated using external resources or the submitted data are valid for the role of registrant.

### General parameters (status)

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| None      | -         | No data returned for this section |

### registrar data section

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| `registrar.reference` | yes | Reference for unique identification of the original request from the registrar |
| `registrar.transactionid` | yes | Registrars transactionid |
| `registrar.token` | yes | Token for inclusion on either mailform or EPP request |

### registrant data section (if validated)

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| `registrant.userid` | yes (A) | Existing user-id, which can be associated with an active user with the registry. This user-id should point to the potential registrant. Equivalent of the mails forms field 4. |
| `registrant.type` | yes (B) | User type, one of: C (company), P (Public Organisation), A (Association) or I (Individual). Equivalent of the mail forms field 4a. |
| `registrant.name` | yes (B) | Company, organization, association or person name, in reference to the above field. Equivalent of mail form field 4b. |
| `registrant.vatnumber` | no (B)* | VAT number, equivalent of mail form 4c. Mandatory for type C (company) and P (public organisation), can be provided for A (association) if the specified association has a VAT number. |
| `registrant.pnumber` | no (B)* | P-number. Mandatory for type C (company) and P (public organisation), can be provided for A (association) if the specified association has a p-number. |
| `registrant.address.street1` | yes (B) | Equivalent of mail form field 4f. |
| `registrant.address.street2` | no (B) | Equivalent of mail form field 4g. |
| `registrant.address.street3` | no (B) | Equivalent of mail form field 4h. |
| `registrant.address.zipcode` | yes (B) | Equivalent of mail form field 4i. |
| `registrant.address.city` | yes (B) | Equivalent of mail form field 4j. |
| `registrant.address.countryregionid` | yes (B) | Two-letter country code based on ISO 3166 Alpha 2. Equivalent of mail form field 4fk. (See references below). |
| `registrant.email` | yes (B) | Equivalent of mail form field 4l. |
| `registrant.phone` | yes (B) | Equivalent of mail form field 4m. |
| `registrant.telefax` | no (B) | Equivalent of mail form field 4n. |

### registrant data section (if not-validated)

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| None      | -         | No data returned for this section |

### domain data section

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| `domain.N.name` | yes | Valid Danish domain name. N indicates a number between 1 and 10. |

## on_reject

This URL is called if the user decides to decline the request, please note that the returned data have been validated in this case or the submitted data are valid for the role of registrant.

### General parameters (status)

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| None      | -         | No data returned for this section |

### registrar data section

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| `registrar.reference` | no | Reference for unique identification of the original request from the registrar |
| `registrar.transactionid` | yes | Registrars transactionid |

### registrant data section

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| None      | -         | No data returned for this section |

### domain data section

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| None      | -         | No data returned for this section |

## on_edit

This URL is called if the user decides to edit the request, please note that the data has not been validated by DK Hostmaster using external resources and should be resubmitted.

### General parameters (status)

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| `token` | yes | Token for inclusion on either mailform or EPP request |
| `status` | yes | Value: `accepted` |

### registrar data section

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| `registrar.reference` | yes | Reference for unique identification of the original request from the registrar |
| `registrar.transactionid` | yes | Registrars transactionid |

## on_fail

This URL is called if the user is unable to validate and all attempts to validate towards external resources are exhausted or the submitted data are not valid for the role of registrant. Please note that the returned data have not been validated and are returned _as-is_.

### General parameters (status)

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| None      | -         | No data returned for this section |

### registrar data section

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| `registrar.reference` | no | Reference for unique identification of the original request from the registrar |
| `registrar.transactionid` | yes | Registrars transactionid |

### registrant data section

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| None      | -         | No data returned for this section |

### domain data section

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| None      | -         | No data returned for this section |

## on_error

This URL is called if the request is valid, but it cannot be presented by the pre-activation service.
If possible and applicable the error location will be attempted identified if it relates to a single field pointing to malformed or missing data. The latter will be described in the error message in the `error` field. The request should be resubmitted if possible.

### General parameters (status)

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| `status` | yes | Value: `error` |
| `error` | yes | Error key |
| `error_text` | yes | Error message |
| `where` | no | Error location indicator if applicable, meaning possibly a missing data field or completely incomprehensible piece of data |

### registrar data section

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| `registrar.reference` | no | Reference for unique identification of the original request from the registrar |
| `registrar.transactionid` | yes | Registrars transactionid |

### registrant data section

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| None      | -         | No data returned for this section |

### domain data section

| Parameter | Mandatory | Description |
| --------- | --------- | ----------- |
| None      | -         | No data returned for this section |

# Validation

Validation of the registrant is a legal requirement outlined in the danish law on domain names.

Validation can be accomplished according to the following matrix.

| Entity | Country | Resource |
| ------ | ------- | -------- |
| Company | DK | CVR |
| Association | DK | CVR |
| Public Organisation | DK | CVR |
| Individual | DK | Name and address, possibly extended to social security number (CPR), hence requiring NemID for collection |
| Company | Non-DK | N/A |
| Association | Non-DK | N/A |
| Public Organisation | Non-DK | N/A |
| Individual | Non-DK | N/A |

Failure to validate according to the above matrix, results in a callback to `on_fail` as specified by the registrar. If the registrar decides to submit the registration application anyway the validation step will be repeated and the registrant will be attempted validated again.

Please note that the validation process is relevant for both new and existing users if a referenced user entity has not been validated on a prior occassion.

# Implementation Limitations
The service comes with some limitations, these are listed here.

## Locale

We only support Danish and English as languages. See: `registrar.language`

## Amount of Domain Names

We only support up to 10 domain names in a single request. If requirements for larger pre-activation requests, this number can be raised.

## Shared-secret Handling

For now the process of handling the shared-secret is handled manually and via DK Hostmaster, please see the technical contact below.

## Encryption

For now DK Hostmaster only support SHA-256.

## Sandbox Environment

The sandbox is for integration and development purposes and will be used to evaluate changes to the protocol prior to deployment in production. The environment comes with some predefined data:

* Registrar-id: REG-999999
* Shared secret: dkhm-sandbox-test-secret
* Key-id (`registrar.keyid` - part of the request): 999888

All additional fields should be specified by the calling client.

### Test Data

In addition to the checksum key, the validation part of the flow is sandboxed. 

This mean you cannot provide arbitrary data to the service, since the sandboxed environment does not connect to _live_ external resources. This is both to prohibit extortion of the service and it's interfaces and to be able to support uniform and repeatable behaviour.

Our sandbox currently supports the following test data.

*Name and address of individual:*

| Resource | Data             | Note |
| -------- | ---------------- | ---- |
| Name     | `Peter Pedal`    | |
| Street   | `Pedelvej 1`     | |
| Zipcode  | `4583`           | |
| City     | `Sjællands Odde` | |

*Social Security Number (CPR):*

| Resource | Data             | Note |
| -------- | ---------------- | ---- |
| CPR      | `2908212555`     | Requires NemID (see below) |

*NemID:*

| Resource | Data           | Note |     
| -------- | -------------- | ---- |
| Login    | `PEDAL30`      | |
| Password | `asasas12`   | |
| Keycard  |                | Can be downloaded from: https://appletk.danid.dk/developers/OtpCard?userid=560977162 |

[General information on the use of the demo keycard](https://appletk.danid.dk/developers/viewstatus.jsp?userid=560977162).

*CVR:*

| Resource | Data           | Note |     
| -------- | -------------- | ---- |
| CVR      | `24210375`     |      |
| Name     | `DK HOSTMASTER A/S` | |

The data set might be extended in the future to support more use-case scenarios.

# References

Here is a list of documents and references used in this document:

* [DK Hostmaster: General Terms and Conditions][General Terms and Conditions]
* [DK Hostmaster: EPP Service Specification][DK Hostmaster EPP Service Specification]
* [DK Hostmaster: Current domain registration form][Current domain registration form]
* [DK Hostmaster: Documentation on the current domain registration form][Documentation on the current domain registration form]
* [ISO 3166: Country codes][ISO 3166: Country codes]
* [RFC 3282: Content-language headers][RFC 3282: Content-language headers]
* [RFC 4634: US Secure Hash Algorithms (SHA and HMAC-SHA)][RFC 4634: US Secure Hash Algorithms (SHA and HMAC-SHA)]

# Resources

A list of resources for DK Hostmaster pre-activation service support is listed below.

## Demo Client

We have made a demo client available as open source under the MIT license. The client is implemented using Perl and Mojolicious and is available on Github:

* [preact-demo-client][preact-demo-client]

## Mailing list

DK Hostmaster operates a mailing list for discussion and inquiries  about the DK Hostmaster pre-activation service. To subscribe to this list, write to the address below and follow the instructions. Please note that the list is for technical discussion only, any issues beyond the technical scope will not be responded to, please send these to the contact issue reporting address below and they will be passed on to the appropriate entities within DK Hostmaster.

* preact-discuss+subscribe@liste.dk-hostmaster.dk

## Issue Reporting

For issue reporting related to this specification, the pre-activation service, sandbox or production environments, please contact us.  You are of course welcome to post these to the mailing list mentioned above, otherwise use the address specified below:

* tech@dk-hostmaster.dk

## Additional Information

More information and the latest revision of this specification are available at [the DK Hostmaster website](https://www.dk-hostmaster.dk/english/tech-notes/pre-activation/).

# Data Sheet

| Environment | URI | Description |
| ----------- | --- | ----- |
| Production | https://preact.dk-hostmaster.dk | Available |
| Sandbox | https://preact-sandbox.dk-hostmaster.dk | Available, currently not being reset with regular intervals. |

[DK Hostmaster EPP Service Specification]: https://github.com/DK-Hostmaster/epp-service-specification

[General Terms and Conditions]: https://www.dk-hostmaster.dk/fileadmin/filer/pdf/generelle_vilkaar/general-conditions.pdf

[preact-screendump]: https://raw.githubusercontent.com/DK-Hostmaster/preactivation-service-specification/master/images/preact-screendump.png

[preact-request-flow-diagram]: https://raw.githubusercontent.com/DK-Hostmaster/preactivation-service-specification/master/images/preact-request-flow-diagram.png

[preact-demo-client]: https://github.com/DK-Hostmaster/preact-demo-client-mojolicious

[ISO 3166: Country codes]: http://www.iso.org/iso/country_codes.htm

[RFC 3282: Content-language headers]: https://tools.ietf.org/html/rfc3282

[RFC 4634: US Secure Hash Algorithms (SHA and HMAC-SHA)]: https://tools.ietf.org/html/rfc4634

[Current domain registration form]: https://www.dk-hostmaster.dk/fileadmin/formularer/dk-5.00en.txt

[Documentation on the current domain registration form]: https://www.dk-hostmaster.dk/english/technical-administration/forms/register-domainname/

