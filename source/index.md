---
title: Open Collaboration Services v2 (WIP)

language_tabs:
  - Examples

toc_footers:
  - <a href='https://github.com/LukasReschke/OpenCloudMeshSpecification'>Send Change Request</a>

search: true
---

# Introduction

This document defines APIs and protocols to enable integration between different web-services, servers and clients.

"OCS v2.0" consists of a set of API endpoints mainly targeted to be implemented by consumers and providers of file storage / sharing servers ("cloud").

The goal is to enable Integration of cloud services, web services and social communities with each other, with desktop and mobile applications. This must be done in a decentralized and federated way, free and secure, privacy protected and vendor independent. OCS aims to solve these problems.


## Notational Conventions
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC2119](https://tools.ietf.org/html/rfc2119).

## Terminology
- **"Cloud":**
    -  Referring to file storage / sharing servers.
- **"Consumer":**
    - A desktop or mobile application or web service that access services that are provided by a server.
- **"Service provider" or "server":**
  - A web service or server who provides a open collaboration services compatible APIs.
- **"Provider service list":** 
  - A JSON configuration endpoint which specifies which services are provided by a server or service provider.
- **"Module" or "service":**
    - OCS is aimed to be a modular system allowing providers to only implement the required subsets. Any functionality is thus encapsulated in so called "modules" or "services" which are all optional.

## Security Considerations
### Transmission layer
All data transfer SHOULD be TLS encrypted to ensure the integrity and security of transferred data.

Consumers MUST properly validate the certificate chain and in case of an error cancel the connection and SHOULD notify the user about the occurred problem.

For enhanced security Consumers MAY follow the Public Key Pinning Extension for HTTP ([RFC 7469](http://www.rfc-editor.org/rfc/rfc7469.txt)) as well as other HTTP security best practices.

### Secrets
Any secrets used to exchange data MUST be generated using a strong random number generator (such as `/dev/urandom`)

## Performance / Scalability
The service must be usable by a lot of users in parallel. Because of that it is important to build the architecture in a scalable way. Every component of the architecture must be cluster enabled, accessible in a parallel way and stateless.

### Cookies
To work together with load-balanced environments consumers SHOULD resend any cookies as defined in [RFC 6265](http://tools.ietf.org/html/rfc6265). As stated in the "Authentication" section any Basic Auth authentication header MUST be resend.

It shall be noted that OCS endpoints MUST behave properly regardless whether cookies are resent or not.

## Formats / Encoding
To avoid interoperability problems this section defines formats and encoding that OCS endpoints as well as consumers MUST obey.

### Encoding
Any data sent and received MUST be UTF-8 encoded.

### Date and time format
All dates and time representations MUST be in [ISO 8601](http://www.iso.org/iso/home/standards/iso8601.htm) format. This means all of the following values are valid:

- Date
  - 2015-06-03
- Combined date and time in UTC
  - 2015-06-03T13:21:58+00:00
  - 2015-06-03T13:21:58Z
- Week
  - 2015-W23
- Date with week number
  - 2015-W23-3
- Ordinal date
  - 2015-154

Consumers as well as endpoints SHALL NOT make assumption about the representation as long as it follows ISO 8601. The preferred variation is though the regular Date representation "YYYY-MM-DD".

If an invalid date format has been provided consumers and endpoints MAY stop processing the data. 


## Output format / OCS Format
OCS endpoints MUST be able to output data in a XML format as well as a JSON representation. This means that the returned XML output SHALL NOT contain any attributes as these cannot be mapped properly to a JSON object.

To specify the encoding a GET parameter `format` can be by the customer send with one of the following values:

- `json`
  - Returns a JSON formatted output, the Content-Type MUST be set to `application/json; charset=utf-8`
- `xml`
  - Returns a XML formatted output, the Content-Type MUST be set to `text/xml; charset=UTF-8`

A OCS response MUST consist of the following elements:

- `ocs`: Array that contains the whole response
    - `meta`: Array that contains meta information
        - `status`: The status of the response, either "ok" or "fail". MUST be "ok" if statuscode is set to 200, "fail" otherwise.
        - `statuscode`: The OCS status code of the response, everything except 200 MUST be handled as failure.
        - `message`: An optional message that MAY contain a status message, such as a error message.
    - `data`: Array that contains the actual response, content of the array depends completely on the endpoint.

This means that a message MUST look like the following for XML responses, be aware that empty fields MUST still be existent:

```xml
<?xml version="1.0"?>
<ocs>
 <meta>
  <status>ok</status>
  <statuscode>200</statuscode>
  <message/>
 </meta>
 <data>
  <users>
   <!-- List of user names -->
   <element>Frank</element>
   <element>admin</element>
   <element>test</element>
   <element>test1</element>
  </users>
 </data>
</ocs>
```

The mapped JSON response MUST look like the following, be aware that empty fields MUST have a value of `null`: 

```json
{
    "ocs": {
        "meta": {
            "status": "ok",
            "statuscode": 200,
            "message": null
        },
        "data": {
            "users": [
                "Frank",
                "admin",
                "test",
                "test1"
            ]
        }
    }
}
```
### Reserved status codes

The following status codes are reserved and MUST not be used for any other purposes. Other status codes indicate custom errors that can be used by modules. 

| Statuscode | Usage                 |
|------------|-----------------------|
| 200        | Success               |
| 401        | Authentication failed |
| 404        | Unknown request       |

Note that all responses MUST return a HTTP status code 200, the OCS status code is completely independent from this. In case of a fatal server error that makes the instance inaccessible a 5xx error MAY be acceptable.

# Authentication

```http
GET /{endpoint} HTTP/1.1
Host: localhost
Authorization: Basic YWRtaW46YWRtaW4=
OCS-REQUEST: true
```

> Make sure to replace `YWRtaW46YWRtaW4=` with the encoded credentials.

Clients SHOULD directly try HTTP Basic Authentication and not try to perform a Digest Authentication based authentication. This will save the amount of requests originating to the endpoint.

Authentication is performed by sending a Basic HTTP Authentication header. Thus OCS endpoints MUST support the Basic Auth Access Authentication system as defined in [RFC2617](https://tools.ietf.org/html/rfc2617). 

The credentials provided in the Authentication header MUST be UTF-8 encoded. Taking `contrasea` as example the correct encoding would be `Y29udHJhc2XDsWE=`, any other encoding such as  ISO-8859-1 (`Y29udHJhc2XxYQ==`) is considered invalid and MUST NOT be used. 

OCS compatible APIs MAY use cookies as defined in [RFC 6265](http://tools.ietf.org/html/rfc6265) to authenticate future requests as well, if after successful authentication the endpoint sends further cookies clients SHOULD resend these for future requests. However, any request MUST then include the Basic Authentication header as well as the provided cookies. When resending cookies consumers MUST add the following HTTP header to all requests: `OCS-REQUEST: true`.

OCS endpoints MUST in all cases handle connections correctly regardless whether cookies are sent.

# Service Discovery
The provider service list are used by OCS for service discovery purposes to allow mapping the provided services and endpoint URIs.

Every service provider MUST provide a provider service list located as `/ocs-provider/` under the directory of the application. For example if there would be an OCS compatible application running at `https://demo.owncloud.org/` the provider service list MUST be located at `https://demo.owncloud.org/ocs-provider/`. If the application would run on `https://mycloud.com/mycloud/` the service MUST be accessible from `https://mycloud.com/mycloud/ocs-provider/`.

Serving the provider service list at this location allows providers to generate them dynamically also in environments where programming languages pose limitations on the routing, such as PHP. In the PHP scenario this would mean that a `index.php` file stored under `/ocs-provider/` would be served.

A provider service list consists of the following elements:

1. `version`: Version identifier
  - Integer indicating the current version of the supported OCS standard. (e.g. `int(2)`)
2. `services`: Array of services
  - A of services served by OCS from this instance. Services are defined as following:
    - `key`: Name of the endpoints
    - `version`: Version of the specific endpoint (may differ from the OCS version)
    - `endpoints`: Array of endpoint names and the relative URI
            - `key`: Name of the endpoint
            - `value`: Relative URI to the endpoint

A provider service definition MUST look like the following (the endpoint URI MAY be changed by providers):

```json
{
    "version": 2,
    "services": {
        "FEDERATED_SHARING": {
            "version": 1,
            "endpoints": {
                "share": "/ocs/cloud/shares",
                "accept": "/ocs/cloud/shares"
            }
        },
        "PRIVATE_DATA": {
            "version": 1,
            "endpoints": {
                "getattribute": "/ocs/privatedata/getattribute",
                "setattribute": "/ocs/privatedata/setattribute",
                "deleteattribute": "/ocs/privatadata/deleteattribute"
            }
        }
    }
}
```

In this case if an instruction of the module `PRIVATE_DATA` serving at `demo.example.org/mycloud/`defines to interact on the endpoint `getattribute` the absolute URI would resolve to `demo.example.org/mycloud/ocs/privatedata/getattribute`.

## Cross-Origin communication

Due to the Same-Origin-Policy it is usually not possible for browsers to access the content of files hosted on another domain. To enable browser-based clients, providers are encouraged to set proper CORS headers on the resource to allow cross-domain access to the provider service list.

If a provider service list cannot be accessed by a consumer, consumers MUST perform the OCS request and in case of a `404` failure consider the endpoint to be not existent.

## Versioning

When declaring a version within the service the service MUST stay backwards compatible on the defined URI. Providers MAY remove the endpoint but SHALL NOT make an existing endpoint incompatible. 

An idiomatic approach to this would be to prepend versions to the URI, such as `/api/1/` for version 1 of an endpoint and `/api/2/` for the backwards incompatible version 2 of an API.

# Modules
OCS is based on modules providing specialized functionalities. A service provider MAY only implement a smaller subset of a defined module.

Providers MUST ensure that only supported modules are listed in the provider service list. Providers MAY not include modules into the providers service list if the endpoints are considered private API.

The current specified modules are:

- **FEDERATED_SHARING:**
  - Allows different cloud service providers to share files over different instances.
- **PRIVATE_DATA:** 
  - Key-Value store allowing consumers to store and retrieve values.
- **SHARING:** 
  - Allows consumers to share local files.
- **PROVISIONING:** 
  - Allows consumers to manage users, groups and applications on the instance.
- **ACTIVITY:** 
  - Activity stream provided by the server

More modules are subject to be added to future OCS versions. Furthermore providers MAY provide their own modules if these get a vendor prefix. (e.g. "owncloud-print" for a fictional printing API of the "ownCloud" product)

# Federated Sharing
> A valid service definition looks as following:

```json
{
    "version": 2,
    "services": {
        "FEDERATED_SHARING": {
            "version": 1,
            "endpoints": {
                "share": "/ocs/v2.php/cloud/shares",
                "webdav": "/public.php/webdav/"
            }
        }
    }
}
```

The "FEDERATED_SHARING" module allows consumers to share files between service providers compatible with this module. This module handles the following tasks:

- Sending a share offer to other providers (`share`)
- Accepting a share offer from other providers (`share`)
- Denying a share offer from other providers (`share`)
- Unsharing a previously shared file (`share`)
- Accessing shared files using WebDAV (`webdav`)

## Sending a share offer

This endpoint takes share offers from remote instances, once the recipient has logged-in he is expected to accept or deny the share.

### HTTP Request
```http
POST /ocs/v2.php/cloud/shares HTTP/1.1
Host: localhost
Content-Length: 119
Content-Type: application/x-www-form-urlencoded

shareWith=RecipientName&token=MyRandomToken&name=Documents&remoteId=5&owner=ShareingUser&remote=http://localhost/master/
```

`POST http://example.com/{share}`

### Query Parameters

Parameter | Default | Description
--------- | ------- | -----------
shareWith |  | User name of the receiving user.
token | | Unique and secret token used to access the file.
name | | Name of the file or folder.
remoteId | | Unique ID to identify the file on the sender side, used for accepting and denying shares.
owner | | User name of the sending user.
remote | | URI of the sending instance.

### Response
> In case of success, a OCS success message without further details is returned:

```xml
<?xml version="1.0"?>
<ocs>
 <meta>
  <status>ok</status>
  <statuscode>100</statuscode>
  <message/>
 </meta>
 <data/>
</ocs>
```

The expected response is, in case of success, a OCS success message. In error cases the following OCS status codes MUST be used:

| Status code | Meaning               |
|-------------|-----------------------|
| 400 | Invalid parameters|
| 500 | Internal server error |
| 503 | If the server does not support Federated Sharing (i.e. disabled by administrator) |

In case of an error consumers SHOULD be made aware of the error. The OCS message MUST be used by the endpoint to specify a proper error message that can be used to analyze issues.

<aside class="notice">
This request needs to be sent to the receiving instance by the sending instance. This request is not expected to be sent directly by an user.
</aside>

## Accept a share offer
After a share offer has been received the receiving instance should notify the user in question and give the possibility to accept or deny a share offer. Using this API call a federated share can be accepted.

Providers MAY inform the sending user if a share has been accepted.

### HTTP Request
> Assuming a provider wants to accept the share request of the file sent in the `share` endpoint (`remoteId: 6`, `token: TjuMOB1as7iZ1c6`) the following request needs to be sent to the remote (`http://sender.example.org`): 

```http
POST /ocs/v2.php/cloud/shares/6/accept HTTP/1.1
Host: sender.example.org
Content-Type: application/x-www-form-urlencoded
Content-Length: 21

token=TjuMOB1as7iZ1c6
```

`POST http://example.com/{share}/{remoteId}/accept`

### URL Parameters
Parameter | Default 
--------- | ------- 
remoteId |   Received `remoteId` of the share.

### Query Parameters

Parameter | Default | Description
--------- | ------- | -----------
token | | Unique and secret token used to access the file.


### Response
> In case of success, a OCS success message without further details is returned:

```xml
<?xml version="1.0"?>
<ocs>
 <meta>
  <status>ok</status>
  <statuscode>100</statuscode>
  <message/>
 </meta>
 <data/>
</ocs>
```

The expected response MUST in any case be a OCS success message even if the deny did not work. The only exception being 503 when Federated Sharing is not enabled.

| Status code | Meaning               |
|-------------|-----------------------|
| 503 | If the server does not support Federated Sharing (i.e. disabled by administrator) |

<aside class="notice">
This request needs to be sent to the sending instance by the receiving instance. This request is not expected to be sent directly by an user.
</aside>

## Reject a share
This endpoint informs the sender that the recipient rejected the share. This endpoint is also intended to be used if the user first accepted the share and later decides to unshare it.

### HTTP Request
> Assuming a provider wants to reject the share request of the file sent in the `share` endpoint (`remoteId: 6`, `token: TjuMOB1as7iZ1c6`) the following request needs to be sent to the remote (`http://sender.example.org`): 

```http
POST /ocs/v2.php/cloud/shares/6/decline HTTP/1.1
Host: sender.example.org
Content-Type: application/x-www-form-urlencoded
Content-Length: 21

token=TjuMOB1as7iZ1c6
```

`POST http://example.com/{share}/{remoteId}/accept`

### URL parameters
Parameter | Default 
--------- | ------- 
remoteId |   Received `remoteId` of the share.

### Query Parameters

Parameter | Default | Description
--------- | ------- | -----------
token | | Unique and secret token used to access the file.


### Response
> In case of success, a OCS success message without further details is returned:

```xml
<?xml version="1.0"?>
<ocs>
 <meta>
  <status>ok</status>
  <statuscode>100</statuscode>
  <message/>
 </meta>
 <data/>
</ocs>
```

The expected response MUST in any case be a OCS success message even if the deny did not work. The only exception being 503 when Federated Sharing is not enabled.

| Status code | Meaning               |
|-------------|-----------------------|
| 503 | If the server does not support Federated Sharing (i.e. disabled by administrator) |

<aside class="notice">
This request needs to be sent to the sending instance by the receiving instance. This request is not expected to be sent directly by an user.
</aside>

## Unshare a file
Allows owners of shared files and folders to notify recipient of a revocation of the access permissions.

<aside class="notice">
Instances MUST be able to handle not accessible remote storages gracefully even when a folder has not been unshared. The unshared endpoint is there to indicate to a remote host that a storage has finally removed and is not just temporarily unavailable.
</aside>

### HTTP Request
> Assuming a provider wants to unshare the share of the file sent to the `share` endpoint (`remoteId: 6`, `token: TjuMOB1as7iZ1c6`) the following request needs to be sent to the receiver (`http://receiver.example.org`): 

```http
POST /ocs/v2.php/cloud/shares/8/unshare?format=json HTTP/1.1
Host: receiver.example.org 
Content-Type: application/x-www-form-urlencoded
Content-Length: 21

token=kW1gR9TRKXW9Jwk
```

`POST http://example.com/{share}/{remoteId}/unshare`

### URL parameters
Parameter | Default 
--------- | ------- 
remoteId |   Received `remoteId` of the share.

### Query Parameters

Parameter | Default | Description
--------- | ------- | -----------
token | | Unique and secret token used to access the file.


### Response
> In case of success, a OCS success message without further details is returned:

```xml
<?xml version="1.0"?>
<ocs>
 <meta>
  <status>ok</status>
  <statuscode>100</statuscode>
  <message/>
 </meta>
 <data/>
</ocs>
```

The expected response MUST in any case be a OCS success message even if the deny did not work. The only exception being 503 when Federated Sharing is not enabled.

| Status code | Meaning               |
|-------------|-----------------------|
| 503 | If the server does not support Federated Sharing (i.e. disabled by administrator) |

<aside class="notice">
This request needs to be sent to the sending instance by the receiving instance. This request is not expected to be sent directly by an user.
</aside>

## Accessing shared files
> To access the file `sharedfile.txt` of the federated share with the token `kW1gR9TRKXW9Jwk` the following request has to be sent. Note that as password the base64 representation of `kW1gR9TRKXW9Jwk:` is used. The ending `:` is required.

```http
GET /public.php/webdav/sharedfile.txt HTTP/1.1
Host: localhost
Authorization: Basic a1cxZ1I5VFJLWFc5SndrOg==
OCS-REQUEST: true
```

Shares files may be accessed using WebDAV ([RFC 4918](https://tools.ietf.org/html/rfc4918)) under the specified `webdav` endpoint using HTTP Basic Authentication.

The credentials will be of the value `token:`, note the empty password field.

# Sharing
> A valid service definition looks as following:

```json
{
    "version": 2,
    "services": {
        "SHARING": {
            "version": 1,
            "endpoints": {
                "share": "/ocs/v2.php/apps/files_sharing/api/v1/shares"
            }
        }
    }
}
```

The `SHARING` module allows to handle file sharing on the same cloud instance. It offers the following functions:

1. Get a list of shares (`share`)
2. Get a list of shared files in a folder (`share`) 
3. Get information about a specific share (`share`)
4. Create a new share (`share`)
5. Delete an existing share (`share`)
6. Update an existing share (`share`)

## Get list of shares
Get a list of all shared files for the currently logged-in user.

### HTTP Request
> Lists all shares of the user admin with the password admin:

```http
GET /master/ocs/v2.php/apps/files_sharing/api/v1/shares HTTP/1.1
Host: localhost
Authorization: Basic YWRtaW46YWRtaW4=
OCS-REQUEST: true
```

`POST http://example.org/{share}`

### Query Parameters

Parameter | Default | Description
--------- | ------- | -----------
shared_with_me | `false` | Whether files shared with the user should get displayed. Defaults to `false`, `true` to only display shares that the user received.
path | '/' | Path to folder, if empty no restriction is set.
reshares | `false` | Whether reshares should get returned.
subfiles | `false` |  Whether all shares within the folder should be returned.

### Response
> In case there are shares a successful OCS response object MUST be returned by the server, the shares MUST be embedded in the data element and at least consist of the following elements:

```xml
<element>
  <!-- Unique identified of the share (integer) -->
  <id>1</id>
  <!-- The item_type either "file" or "folder" (string) -->
  <item_type>file</item_type>
  <!-- '0' = user; '1' = group; '3' = public link (integer) -->
  <share_type>2</share_type>
   <!-- In case when shared with a user or a group the name (string) -->
  <share_with>Rachel</share_with>
  <!-- Path where the share is located (string) -->
  <path>/sharedFile.txt</path>
  <!-- Permissions: 1 = read; 2 = update; 4 = create; 8 = delete; 16 = share; 31 = all (default: 31, for public shares: 1) (integer) -->
  <permissions>1</permissions>
  <!-- Date when the share should expire (ISO 8601) -->
  <expiration>2015-06-12</expiration>
  <!-- Unique token that may be used to access the file in case of a public link (string) -->
  <token></token>
  <!-- Unique token that may be used to access the file in case of a public link (string) -->
  <uid_owner>Oscar</uid_owner>
  <!-- Human readable name, may be chosen by the user itself (string) -->
  <displayname_owner>Oscar Meyer</displayname_owner>
</element>
```

The expected response MUST in any case be a OCS success message containing the share data. 

| Status code | Meaning               |
|-------------|-----------------------|
| 400 | Not a directory (if the `subfile` argument was used) |
| 401 | Authentication was not successful. |
| 404 | User has no shared files or folder does not exist. |

## Get information about a share
Get information about a specific share using the share id.

### HTTP Request
> Following request would request information about the share with the ID "1" under the context of the user "admin".

```http
GET /master/ocs/v2.php/apps/files_sharing/api/v1/shares/1 HTTP/1.1
Host: localhost
Authorization: Basic YWRtaW46YWRtaW4=
OCS-REQUEST: true

```

`POST http://example.com/{share}/{shareId}`

### URL Parameters

Parameter | Default | Description
--------- | ------- | -----------
shareId |  | ID of the requested share.

### Response
> In case of success, a OCS success message with the following data structure is returned:

```xml
<element>
  <!-- Unique identified of the share (integer) -->
  <id>1</id>
  <!-- The item_type either "file" or "folder" (string) -->
  <item_type>file</item_type>
  <!-- '0' = user; '1' = group; '3' = public link (integer) -->
  <share_type>2</share_type>
   <!-- In case when shared with a user or a group the name (string) -->
  <share_with>Rachel</share_with>
  <!-- Path where the share is located (string) -->
  <path>/sharedFile.txt</path>
  <!-- Permissions: 1 = read; 2 = update; 4 = create; 8 = delete; 16 = share; 31 = all (default: 31, for public shares: 1) (integer) -->
  <permissions>1</permissions>
  <!-- Date when the share should expire (ISO 8601) -->
  <expiration>2015-06-12</expiration>
  <!-- Unique token that may be used to access the file in case of a public link (string) -->
  <token></token>
  <!-- Unique token that may be used to access the file in case of a public link (string) -->
  <uid_owner>Oscar</uid_owner>
  <!-- Human readable name, may be chosen by the user itself (string) -->
  <displayname_owner>Oscar Meyer</displayname_owner>
</element>
```

The expected response is, in case of success, a OCS success message containing the data structure. In error cases the following OCS status codes MUST be used:

| Status code | Meaning               |
|-------------|-----------------------|
| 401 | Authentication was not successful |
| 404 | Share does not exist. |

In case of an error consumers SHOULD be made aware of the error. The OCS message MUST be used by the endpoint to specify a proper error message that can be used to analyze issues.

## Create a new share
Shares a file or folder with an user on the same instance.

### HTTP Request
> Following request will share the file `/welcome.txt` of the user `admin` with the user `test`:

```http
POST /master/ocs/v2.php/apps/files_sharing/api/v1/shares HTTP/1.1
Host: localhost
Authorization: Basic YWRtaW46YWRtaW4=
OCS-REQUEST: true
Content-Length: 44
Content-Type: application/x-www-form-urlencoded

path=/welcome.txt&shareType=0&shareWith=test
```

`POST http://example.com/{share}`


### Query Parameters

Parameter | Default | Description
--------- | ------- | -----------
path |  | Absolute path to the file/folder which should be shared.
shareType | | 0 = user; 1 = group; 3 = public link
shareWith | | User or group id with which the file or folder should be shared.
publicUpload |`false`| Whether to allow public upload to a public shared folder.
password | | Password to protect a publicly shared file with.
permissions |31|1 = read; 2 = update; 4 = create; 8 = delete; 16 = share; 31 = all

### Response
> In case the share has been created a successful OCS response object MUST be returned by the server, the shares MUST be embedded in the data element and at least consist of the following elements:

```xml
<element>
  <!-- Unique identified of the share (integer) -->
  <id>1</id>
  <!-- The item_type either "file" or "folder" (string) -->
  <item_type>file</item_type>
  <!-- '0' = user; '1' = group; '3' = public link (integer) -->
  <share_type>2</share_type>
   <!-- In case when shared with a user or a group the name (string) -->
  <share_with>Rachel</share_with>
  <!-- Path where the share is located (string) -->
  <path>/sharedFile.txt</path>
  <!-- Permissions: 1 = read; 2 = update; 4 = create; 8 = delete; 16 = share; 31 = all (default: 31, for public shares: 1) (integer) -->
  <permissions>1</permissions>
  <!-- Date when the share should expire (ISO 8601) -->
  <expiration>2015-06-12</expiration>
  <!-- Unique token that may be used to access the file in case of a public link (string) -->
  <token></token>
  <!-- Unique token that may be used to access the file in case of a public link (string) -->
  <uid_owner>Oscar</uid_owner>
  <!-- Human readable name, may be chosen by the user itself (string) -->
  <displayname_owner>Oscar Meyer</displayname_owner>
</element>
```

The expected response is, in case of success, a OCS success message with the defined data structure. In error cases the following OCS status codes MUST be used:

| Status code | Meaning               |
|-------------|-----------------------|
| 400 | Unknown share type.|
| 401 | Authentication was not successful.|
| 404 | File couldnt get shared.|

## Unshare an existing share
Unshares a shared file or folder.

### HTTP Request
> Following request will delete the share with the id `1` of the user `admin`:

```http
DELETE /master/ocs/v2.php/apps/files_sharing/api/v1/shares/1 HTTP/1.1
Host: localhost
Authorization: Basic YWRtaW46YWRtaW4=
OCS-REQUEST: true
```

`DELETE http://example.com/{share}/{shareId}`

### URL Parameters

Parameter | Default | Description
--------- | ------- | -----------
shareId |  | ID of the share to revoke.

### Response
```xml
<?xml version="1.0"?>
<ocs>
 <meta>
  <status>ok</status>
  <statuscode>100</statuscode>
  <message/>
 </meta>
 <data/>
</ocs>
```

The expected response is, in case of success, a OCS success message with the defined data structure. In error cases the following OCS status codes MUST be used:

| Status code | Meaning               |
|-------------|-----------------------|
| 401 | Authentication was not successful.|
| 404 | Share could not get unshared.|

## Update an existing share
Updates an existing share such as adjusting the permissions or the password.

### HTTP Request
> Following request will set the permissions of the share with the id `9` to `31:

```http
PUT /master/ocs/v2.php/apps/files_sharing/api/v1/shares/9 HTTP/1.1
Host: localhost
Authorization: Basic YWRtaW46YWRtaW4=
OCS-REQUEST: true
Content-Length: 14
Content-Type: application/x-www-form-urlencoded

permissions=31
```

`PUT http://example.com/{share}/{shareId}`

### URL Parameters

Parameter | Default | Description
--------- | ------- | -----------
shareId || ID of the share to update.

### Query Parameters

Parameter | Default | Description
--------- | ------- | -----------
publicUpload |`false`| Whether to allow public upload to a public shared folder 
password ||Password to protect a publicly shared file with
permissions ||1 = read; 2 = update; 4 = create; 8 = delete; 16 = share; 31 = all 

### Response
```xml
<?xml version="1.0"?>
<ocs>
 <meta>
  <status>ok</status>
  <statuscode>100</statuscode>
  <message/>
 </meta>
 <data/>
</ocs>
```

The expected response is, in case of success, a OCS success message with the defined data structure. In error cases the following OCS status codes MUST be used:

| Status code | Meaning               |
|-------------|-----------------------|
| 400 | Wrong or no update parameter given.|
| 401 | Authentication was not successful.|
| 403 | Public upload disabled by the admin.|
| 404 | Could not update share.|

# Activity
> A valid service definition looks as following:

```json
{
    "version": 2,
    "services": {
        "ACTIVITY": {
            "version": 1,
            "endpoints": {
                "list": "/ocs/v2.php/cloud/activity"
            }
        }
    }
}
```

The "ACTIVITY" module allows consumers  to show a list of actions happening on the cloud service. This can for example be a collection of recently created or deleted files. This module has only a `list` endpoint used to gather this list.


### HTTP Request
> Following request would request the data of the user `admin`:
```http
GET /ocs/v2.php/cloud/activity HTTP/1.1
Host: localhost
Authorization: Basic YWRtaW46YWRtaW4=
OCS-REQUEST: true
```

`POST http://example.com/{list}`

### Query Parameters

Parameter | Default | Description
--------- | ------- | -----------
start|0|Optional value with which event to start.
count|30|Optional value on how many values should get returned. 

### Response
```xml
<?xml version="1.0"?>
<ocs>
 <meta>
  <status>ok</status>
  <statuscode>200</statuscode>
  <message/>
 </meta>
 <data>
  <element>
   <!-- Unique identifier for the event -->
   <id>2</id>
   <!-- Subject of the mes-->
   <subject>You deleted ownCloudUserManual.pdf</subject>
   <!-- More descriptive text of the action (optional) -->
   <message></message>
   <!-- The affected file -->
   <file>/ownCloudUserManual.pdf</file>
   <!-- Link to the file or folder -->
   <link>http://example.org/index.php/apps/files?dir=%2F</link>
   <!-- Timestamp when the event happened -->
   <date>2015-06-10T09:42:58+00:00</date>
  </element>
  <element>
   <id>1</id>
   <subject>You created test.txt</subject>
   <message></message>
   <file>/test.txt</file>
   <link>http://example.org/index.php/apps/files?dir=%2F</link>
   <date>2015-06-10T09:22:38+00:00</date>
  </element>
 </data>
</ocs>
```
The response MUST be a OCS success message or a 993 forbidden statuscode if the user is not authenticated. In case of a success the response must contain the defined data blob.

In case a start or count parameter is specified endpoints MUST return the newest entries first to allow consumers to chunk the event transmission.

# PROVISIONING [Experimental]

The "PROVISONING" module allows management of users and groups on a service provider. Specifically it supports the following functionalities:

1. Managing users (`user`)
    - Create users
    - Edit users
    - Delete users
    - List users
2. Managing groups (`groups`)
    - Create groups
    - Manage subadmin privileges
    - Manage group members
    - Delete groups
    - List groups
3. Managing applications (`apps`)
    - Enable applications
    - Delete applications
    - List applications

A provider service definition MUST look like the following (the endpoint URI MAY be changed by providers):

```json
{
    "version": 2,
    "services": {
        "PROVISIONING": {
            "version": 1,
            "endpoints": {
                "user": "/ocs/v2.php/cloud/users",
                "groups": "/ocs/v2.php/cloud/groups",
                "apps": "/ocs/v2.php/cloud/apps"
            }
        }
    }
}
```

This module is likely to require administrative privileges for accessing and service providers SHOULD review their implementation carefully.

### Create user
Creates a new user on the server.

#### Request
- URL structure: `example.org/{user}`
- HTTP method: POST
- Scope: Authenticated
- Required arguments:
    - `userid`: Username to create (string) 
    - `password`: Password of the user (string)

#### Response
##### Errors
- 400: invalid input data
- 401: authentication was not successful
- 409: username already exists
- 500: unknown error occurred whilst adding the user

##### Success
In case the user has been created successfully a successful OCS response object MUST be returned by the server.
The status code for a successful operation is 201/Created.

### Get users
Retrieves a list of users on the server.

#### Request
- URL structure: `example.org/{user}`
- HTTP method: GET
- Scope: Authenticated
- Optional arguments:
    - `search`: search string (string)
    - `limit`: limit value (int)
    - `offset`: offset value (int)

#### Response
##### Errors
- 401: authentication was not successful

##### Success
If the action could be performed a successful OCS response object MUST be returned by the server. The `data` element MUST contain a list of users such as:

```xml
<?xml version="1.0"?>
<ocs>
 <meta>
  <status>ok</status>
  <statuscode>200</statuscode>
  <message/>
 </meta>
 <data>
  <users>
   <!-- List of users on the instance -->
   <element>Frank</element>
   <element>admin</element>
   <element>test</element>
   <element>test1</element>
  </users>
 </data>
</ocs>
```

### Get a user

Retrieves information about a single user.

#### Request
- URL structure: `example.org/{user}/{uid}`
- HTTP method: GET
- Scope: Authenticated
- URL parameters:
    - `uid`: Identifier of the user to get (string)
- Optional arguments:
    - `search`: search string (string)
    - `limit`: limit value (int)
    - `offset`: offset value (int)

#### Response
##### Errors
- 401: authentication was not successful
- 404: user not found

##### Success
In case the action was successfully a successful OCS response object MUST be returned by the server. The `data` element MUST contain a blob as following, values MAY be empty or more values MAY get returned:

```xml
<?xml version="1.0"?>
<ocs>
 <meta>
  <status>ok</status>
  <statuscode>200</statuscode>
  <message/>
 </meta>
 <data>
  <!-- Email address of the user (string) -->
  <email>user@example.org</email>
  <!-- Whether the user is enabled (boolean) -->
  <enabled>true</enabled>
  <quota>
   <!-- Free storage in byte (int) -->
   <free>80</free>
   <!-- Used storage in byte (int) -->
   <used>20</used>
   <!-- Total storage, free + used in byte (int) -->
   <total>200</total>
   <!-- Percentage of used storage (int) -->
   <relative>20</relative>
  </quota>
  <!-- Human displayable username, may be chosen by the user (string) -->
  <displayname>Hans User</displayname>
 </data>
</ocs>
```

### Edit user attributes
Edits attributes related to a user. Users are able to edit email, displayname and password; admins can also edit the quota value.

#### Request
- URL structure: `example.org/{user}/{uid}`
- HTTP method: PUT
- Scope: Authenticated
- URL parameters:
    - `uid`: Identifier of the user to edit (string)
- Required arguments:
    - `key`: te field to edit (email, quota, displayname, password) (string)
    - `value`: the new value for the field (mixed)

#### Response
##### Errors
- 400: invalid input data
- 400: quote is not a valid value
- 401: authentication was not successful or invalid key
- 404: user not found

##### Success
In case the action was successfully a successful OCS response object MUST be returned by the server.

### Delete user
Deletes a user from the instance.

#### Request
- URL structure: `example.org/{user}/{uid}`
- HTTP method: GET
- Scope: Authenticated
- URL parameters:
    - `uid`: Identifier of the user to delete (string)

#### Response
##### Errors
- 401: authentication was not successful or user is not permitted to perform this action

##### Success
In case the action was successfully a successful OCS response object MUST be returned by the server.
According to general requirements of DELETE being an idempotent operation success is even reported in case the user does not exist.
The status code in case of a successful operation is 204/No Content.

### Get group memberships
Retrieves a list of groups the specified user is member of.

#### Request
- URL structure: `example.org/{user}/{uid}/groups`
- HTTP method: GET
- Scope: Authenticated
- URL parameters:
    - `uid`: Identifier of the user to delete (string)

#### Response
##### Errors
- 401: authentication was not successful or user is not permitted to perform this action

##### Success
If the action could be performed a successful OCS response object MUST be returned by the server. The `data` element MUST contain a list of users such as:

```xml
<?xml version="1.0"?>
<ocs>
  <meta>
    <statuscode>200</statuscode>
    <status>ok</status>
  </meta>
  <data>
    <groups>
      <!-- List of group memberships -->
      <element>admin</element>
      <element>group1</element>
    </groups>
  </data>
</ocs>
```

### Add user to group
Adds the specified user to the specified group.

#### Request
- URL structure: `example.org/{user}/{uid}/groups`
- HTTP method: POST
- Scope: Authenticated
- URL parameters:
    - `uid`: Identifier of the user to add to a group (string)
- Required arguments:
    - `groupid`: Group Id that the user should get added to (string)

#### Response
##### Errors
- 200: successful
- 400: no group specified
- 400: group does not exist
- 400: user does not exist
- 400: failed to add user to group
- 401: authentication was not successful

##### Success
If the action could be performed a successful OCS response object MUST be returned by the server.
The status code for a successful operation is 201/Created.

### Remove user from group
Removes the specified user to the specified group.

#### Request
- URL structure: `example.org/{user}/{uid}/groups`
- HTTP method: DELETE
- Scope: Authenticated
- URL parameters:
    - `uid`: Identifier of the user to add to a group (string)
- Required parameters:
    - `groupid`: Group Id to remove the membership from (string)

#### Response
##### Errors
- 200: successful
- 400: no group specified
- 400: group doesn't exist
- 401: authentication was not successful
- 403: insufficient privileges
- 500: failed to remove user from group

##### Success
In case the action was successfully a successful OCS response object MUST be returned by the server.
According to general requirements of DELETE being an idempotent operation success is even reported in case the user does not exist.
The status code in case of a successful operation is 204/No Content.

### Promote user to subadmin of group
Makes a user the subadmin of a group. Authentication is done by sending a Basic HTTP Authorization header.

#### Request
- URL structure: `example.org/{user}/{uid}/subadmin`
- HTTP method: POST
- Scope: Authenticated
- URL parameters:
    - `uid`: Identifier of the user to promote (string)
- Required parameters:
    - `groupid`: Group id of which the user shall get subadmin privileges (string)

#### Response
##### Errors
- 201: successful
- 404: user does not exist
- 404: group does not exist
- 401: authentication was not successful
- 500: unknown failure

##### Success
If the action could be performed a successful OCS response object MUST be returned by the server.

### Remove subadmin privileges of user
Removes the subadmin rights for the user specified from the group specified. Authentication is done by sending a Basic HTTP Authorization header.

#### Request
- URL structure: `example.org/{user}/{uid}/subadmin`
- HTTP method: DELETE
- Scope: Authenticated
- URL parameters:
    - `uid`: Identifier of the user to demote (string)
- Required parameters:
    - `groupid`: Group id of which the user shall get removed the subadmin privileges (string)

#### Response
##### Errors
- 204: successful
- 401: authentication was not successful
- 500: unknown failure

##### Success
If the action could be performed a successful OCS response object MUST be returned by the server.
According to general requirements of DELETE being an idempotent operation success is even reported in case the user does not exist.
The status code in case of a successful operation is 204/No Content.

### Get subadmin privileges of user
Returns the groups in which the user is a subadmin. Authentication is done by sending a Basic HTTP Authorization header.

#### Request
- URL structure: `example.org/{user}/{uid}/subadmin`
- HTTP method: GET
- Scope: Authenticated
- URL parameters:
    - `uid`: Identifier of the user to get information from (string)

#### Response
##### Errors
- 200: successful
- 401: authentication was not successful
- 404: user does not exist
- 500: unknown failure

##### Success
In case the action was successfully a successful OCS response object MUST be returned by the server. The `data` element MUST contain a list of groups such as:

```xml
<?xml version="1.0"?>
<ocs>
  <meta>
      <status>ok</status>
      <statuscode>200</statuscode>
    <message/>
  </meta>
  <data>
    <!-- List of groups the user is subadmin of -->
    <element>testgroup</element>
  </data>
</ocs>
```

### Get groups on the server
Retrieves a list of groups from the cloud server. Authentication is done by sending a Basic HTTP Authorization header.

#### Request
- URL structure: `example.org/{groups}`
- HTTP method: GET
- Scope: Authenticated
- Optional arguments:
    - `search`:  search string (string)
    - `limit`: limit value (int)
    - `offset`: offset value (int)

#### Response
##### Errors
- 401: authentication was not successful

##### Success
In case the action was successfully a successful OCS response object MUST be returned by the server. The `data` element MUST contain a list of groups such as:

```xml
<?xml version="1.0"?>
<ocs>
  <meta>
    <statuscode>200</statuscode>
    <status>ok</status>
  </meta>
  <data>
    <groups>
      <!-- List of groups on the server -->
      <element>admin</element>
      <element>Support staff</element>
    </groups>
  </data>
</ocs>
```

### Create a new group
Creates a new group. Authentication is done by sending a Basic HTTP Authorization header.

#### Request
- URL structure: `example.org/{groups}`
- HTTP method: POST
- Scope: Authenticated
- Required arguments:
    - `groupid`: identifier of the to creating group (string)

#### Response
##### Errors
- 201: successful
- 400: invalid input data
- 401: authentication was not successful
- 409: group already exists
- 500: failed to add the group

##### Success
If the action could be performed a successful OCS response object MUST be returned by the server.

### Get members of a group
Retrieves a list of group members. Authentication is done by sending a Basic HTTP Authorization header.

#### Request
- URL structure: `example.org/{groups}/{groupId}`
- HTTP method: GET
- Scope: Authenticated
- URL parameters:
    - `groupId`: identifier of group (string)

#### Response
##### Errors
- 401: authentication was not successful

##### Success
If the action could be performed a successful OCS response object MUST be returned by the server. The `data` element MUST contain a list of groups such as:

```xml
<?xml version="1.0"?>
<ocs>
  <meta>
    <statuscode>200</statuscode>
    <status>ok</status>
  </meta>
  <data>
    <!-- List of groups on the server -->
    <users>
      <element>Frank</element>
    </users>
  </data>
</ocs>
```

### Get subadmins of a group
Returns subadmins of the group. Authentication is done by sending a Basic HTTP Authorization header.

#### Request
- URL structure: `example.org/{groups}/{groupId}/subadmins`
- HTTP method: GET
- Scope: Authenticated
- URL parameters:
    - `groupId`: identifier of group (string)

#### Response
##### Errors
- 401: authentication was not successful
- 404: group does not exist
- 500: unknown failure

##### Success
If the action could be performed a successful OCS response object MUST be returned by the server. The `data` element MUST contain a list of subadmins such as:

```xml
<?xml version="1.0"?>
<ocs>
  <meta>
    <statuscode>200</statuscode>
    <status>ok</status>
  </meta>
  <data>
    <users>
      <!-- List of subadmins on the server  -->
      <element>Tom</element>
    </users>
  </data>
</ocs>
```

### Delete a group
Removes a group. Authentication is done by sending a Basic HTTP Authorization header.

#### Request
- URL structure: `example.org/{groups}/{groupId}`
- HTTP method: DELETE
- Scope: Authenticated
- URL parameters:
    - `groupId`: identifier of group (string)

#### Response
##### Errors
- 401: authentication was not successful
- 500: failed to delete group

##### Success
If the action could be performed a successful OCS response object MUST be returned by the server.
According to general requirements of DELETE being an idempotent operation success is even reported in case the user does not exist.
The status code in case of a successful operation is 204/No Content.

### Get list of installed apps
Returns a list of apps installed on the cloud server. Authentication is done by sending a Basic HTTP Authorization header.

#### Request
- URL structure: `example.org/{apps}`
- HTTP method: GET
- Scope: Authenticated

#### Response
##### Errors
- 400: invalid input data
- 401: authentication was not successful

##### Success
If the action could be performed a successful OCS response object MUST be returned by the server. The `data` element MUST contain a list of subadmins such as:

```xml
<?xml version="1.0"?>
<ocs>
  <meta>
    <statuscode>200</statuscode>
    <status>ok</status>
  </meta>
  <data>
    <apps>
      <!-- List of apps on the server  -->
      <element>files</element>
      <element>provisioning_api</element>
    </apps>
  </data>
</ocs>
```

###Get application information
Provides information about a specific application. Authentication is done by sending a Basic HTTP Authorization header.

#### Request
- URL structure: `example.org/{apps}/{appId}`
- HTTP method: GET
- Scope: Authenticated
- URL parameters:
    - `appId`: identifier of app to request information from (string)

#### Response
##### Errors
- 401: authentication was not successful
- 404: application not found

##### Success
In case the action was successfully a successful OCS response object MUST be returned by the server. The `data` element MUST contain a list of information about the app, including at least:

```xml
<?xml version="1.0"?>
<ocs>
  <meta>
    <statuscode>200</statuscode>
    <status>ok</status>
  </meta>
  <data>
    <!-- Id of the application -->
    <id>files</id>
    <!-- Name of the application -->
    <name>Files</name>
    <!-- Description of an application -->
    <description>File Management</description>
  </data>
</ocs>
```

###Enable application
Enable an app. Authentication is done by sending a Basic HTTP Authorization header.

#### Request
- URL structure: `example.org/{apps}/{appId}`
- HTTP method: POST
- Scope: Authenticated
- URL parameters:
    - `appId`: identifier of app to enable (string)

#### Response
##### Errors
- 401: authentication was not successful

##### Success
In case the action was successfully a successful OCS response object MUST be returned by the server.

### Disable application
Disables the specified app. Authentication is done by sending a Basic HTTP Authorization header.

#### Request
- URL structure: `example.org/{apps}/{appId}`
- HTTP method: DELETE
- Scope: Authenticated
- URL parameters:
    - `appId`: identifier of app to delete (string)

#### Response
##### Errors
- 401: authentication was not successful

##### Success
In case the action was successfully a successful OCS response object MUST be returned by the server.
According to general requirements of DELETE being an idempotent operation success is even reported in case the user does not exist.
The status code in case of a successful operation is 204/No Content.

Syntax: ocs/v2.php/cloud/apps/{appid}


# Private Data [Experimental]
> A valid service definition looks as following:

```json
{
    "version": 2,
    "services": {
        "PRIVATE_DATA": {
            "version": 1,
            "endpoints": {
                "store": "/ocs/v2.php/privatedata/setattribute",
                "read": "/ocs/v2.php/privatedata/getattribute",
                "delete": "/ocs/v2.php/privatedata/deleteattribute"
            }
        }
    }
}
```

The `PRIVATE_DATA` module allows consumers to store data in a key-value storage. The store supports multiple sub-stores. This module handles the following tasks:

1. Storing values in the store ("store")
2. Reading data from the store ("read")
3. Deleting data from the store ("delete")

In case of a server error or connection problem consumers MUST handle this gracefully.

## Set value
Sets a value in the key-value store for the currently logged-in user.

#### Request
- URL structure: `example.org/{store}/{storeName}/{key}`
- HTTP method: POST
- Scope: Authenticated
- Arguments:
- URL parameters:
    - `storeName`: Name of the store (string)
    - `key`: Name of they key (string)
- Required arguments:
    - `value`: Value that should get stored (string)

##### Success
The response MUST be a OCS success message.

### Get all values of store
Get all key-values stored within a specific store for the currently logged-in user.

#### Request
- URL structure: `example.org/{store}/{storeName}/{key}`
- HTTP method: GET
- Scope: Authenticated
- URL parameters:
    - `storeName`: Name of the store (string)
    - `key`: Name of they key (string)

#### Response
##### Error
- 401: authentication was not successful

##### Success
The response MUST be a OCS success message and MUST contain a `data` array containing the elements, for example:

```xml
<?xml version="1.0"?>
<ocs>
 <meta>
  <status>ok</status>
  <statuscode>200</statuscode>
  <message/>
 </meta>
 <data>
  <!-- List of stored values -->
  <element>
   <!-- Key of the stored value -->
   <key>MyKey</key>
   <!-- Store in which the data is stored, internally represented as "app" -->
   <app>MyStore</app>
   <!-- Value to store -->   
   <value>ValueToStore</value>
  </element>
  <element>
   <key>AnotherKey</key>
   <app>MyStore</app>
   <value>AnotherValue</value>
  </element>
 </data>
</ocs>
```

In case a store does not contain any values providers MUST return an empty data representation but MUST NOT return any error.

## Get value
Get the value of a key stored within a specific store for the currently logged-in user.

#### Request
- URL structure: `example.org/{read}/{storeName}`
- HTTP method: GET
- Scope: Authenticated
- URL parameters:
    - `storeName`: Name of the store (string)

#### Response
##### Error
- 401: authentication was not successful

##### Success
The response MUST be a OCS success message and MUST contain a `data` array containing the elements, for example:

```xml
<?xml version="1.0"?>
<ocs>
 <meta>
  <status>ok</status>
  <statuscode>200</statuscode>
  <message/>
 </meta>
 <data>
  <element>
   <!-- Key of the stored value -->
   <key>MyKey</key>
   <!-- Store in which the data is stored, internally represented as "app" -->
   <app>MyStore</app>
   <!-- Value to store -->   
   <value>ValueToStore</value>
  </element>
 </data>
</ocs>
```

In case a key does not contain a value providers MUST return an empty data representation but MUST NOT return any error.

## Delete entry
Get the value of a key stored within a specific store for the currently logged-in user.

#### Request
- URL structure: `example.org/{delete}/{storeName}/{key}`
- HTTP method: GET
- Scope: Authenticated
- URL parameters:
    - `storeName`: Name of the store (string)
    - `key`: Name of they key (string)

#### Response
##### Error
- 401: authentication was not successful

##### Success
The response MUST be a OCS success message.


# Errors & Status codes

OCS uses the following error codes returned as HTTP statuscode. The following status codes are reserved and MUST not be used for any other purposes. Other status codes indicate custom errors that can be used by modules. 

| Statuscode | Meaning               |
|------------|-----------------------|
| 200        | Success               |
| 401        | Authentication failed |
| 404        | Unknown request       |
