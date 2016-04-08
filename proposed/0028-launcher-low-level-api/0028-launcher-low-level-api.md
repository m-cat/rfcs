- Feature Name: Exposing Low level APIs for Structured Data and Immutable Data handling
- Type New Feature
- Related components safe_launcher, safe_ffi, safe_core
- Start Date: 06-04-2016
- RFC PR: (leave this empty)
- Issue number: (leave this empty)

# Summary

Proposal for exposing low level APIs from Launcher that will allow app devs to
use Structured Data and Immutable data to create their own topologies.

# Motivation

Access to Structured Data and raw data will be needed for third party applications,
which will allow devs to create their own data structures and also to store raw data.

# Detailed Design

NFS provides a confined structure for directory and file handling, but this
topology may not be the practical data structures that applications need
in real time. Hence, exposing the low level APIs will allow app devs to use
Structured Data to create and manage their own data structures to build applications.

To access the low level APIs, the application must request `LOW_LEVEL_ACCESS`
permission at the time of authorisation with the Launcher.

## Structured Data

Structured Data can be used to reference data in the network using an ID and the `tag_type`.
The ID of the Structured Data is a u8 array of length 64 [u8;64] and the `tag_type` value
can be between the range 10,001 to (2^64 - 1).

|tag_type |Operation|Description|
|---------|---------|-----------|
|9| Encrypted| Encrypted Structured Data read and modified only by owner.|
|10| Encrypted & Versioned| Version enabled encrypted Structured Data read and modified only by owner.|
|11| NotEncypted/Plain |Structured Data for public read but modified only by owner.|
|12| NotEncypted/Plain & Versioned|Version enabled Structured Data for public read but modified only by owner.|

These tag types will make use of the standard implementation of the Structured Data operations in the
[safe_core](https://github.com/maidsafe/safe_core/tree/master/src/core/structured_data_operations).

At this point, `tag_type between the range 10,001 to (2^64-1) and 9-11` will be permitted by the Launcher.
If any specific tag type within the reserved range has to be exposed then it can
also be added later to the permitted range list for the `tag_type` in the Launcher API.

The Structured Data has a size restriction of 100kb. The default implementation in the safe_core
for Structured Data will handle the scenarios even if the size is larger than the allowed size
limit. So the devs using the standard tag types will not have to bother about the size restriction.

If the devs decide to use a more efficient approach than the default implementation,
then they can create a tag_type in the non reserved range between (10001 and 2^64-1) and call the APIs. If a custom tag type is used, then the size restriction should be handled by the application. If the size is more than the permitted size, then a 413 (payload too large) HTTP status code will be returned.

If the tag type is within the custom range (10001 - 2^64-1) then the data wont be encrypted and will be saved as is. It becomes the app devs responsibility to encrypt.

### Versioned Structured Data

Versioned Structured Data will have a list of versions corresponding to the modifications
that have been made. Based on a version ID a specific version of the Structured Data can
be retrieved from the network. Unversioned Structured Data will only return the latest copy.

### Rest API

#### Create

##### Request

###### End point
```
/structuredData
```

###### Method
```
POST
```

###### Body
```javascript
{
  "id": base64 string // [u8;64] array of u8's of length 64 as a base64 String
  "tagType": U64 // Within the permitted range
  "data": base64 // Data that has to be stored as a base64 string    
}
```
|Field| Description|
|-----|------------|
|id   | u8 array of length 64 as a base64 string.|
|tagType| Must be a permitted u64 value (9 - 11 or 10001 - (2^64 - 1)).|
|data| Data to be saved in the Structured Data as base 64 string.|

##### Response

###### Header
```
Status: 200 Ok
```

#### List versions

Retrieve the list of versions for the Structured Data. This will work only for version
enabled tag types (10 & 12), otherwise a 400 (Bad Request) will be thrown. On success,
an available version ID list will be returned. 404 (Not Found) will be returned if the
Structured Data for the specified id and tag type is not found.

##### Request

###### End point
```
/structuredData/versions/{id}/{tagType}
```
|Field|Description|
|-----|-----------|
|id|Structured Data Id as base64 string.|
|tagType| tagType of the Structured Data (10 or 12).|

###### Method
```
GET
```
##### Response

###### Header
```
Status: 200 Ok
```

###### Response Body
```javascript
[ 'id_v1', 'id_v2' ]
```

#### Get
Retrieves the data held by the Structured Data. When a Structured Data is retrieved,
the header will contain a `sd-version` field with a value.

If the user tries to update an older version of the Structured Data - based upon the
`sd-version` value passed while updating will be used to validate the version and a
409 (Conflict) HTTP Status Code will be returned.

In the case of the versioned Structured Data, the `sd-version` will be a base64 string representing the version id.
For the Unversioned Structured Data the `sd-version` will be a u64 number which will refer to the [version field in the Structured Data](https://github.com/maidsafe/rfcs/blob/master/implemented/0000-Unified-structured-data/0000-Unified-structured-data.md#structureddata)

##### Request

###### End point
```
/structuredData/{id}/{tagType}
```
|Field| Description|
|-----|------------|
|id   | u8 array of length 64 as a base64 string.|
|tagType| Must be a permitted u64 value (9 - 11 or 10001 - (2^64 - 1)).|
|data| Data to be saved in the Structured Data as a base64 string.|

###### Method
```
GET
```
##### Response

###### Header
```
Status: 200 Ok
sd-version: {version-reference}
```

###### Body
JSON as base64 string
```
Data held by the Structured Data as a base64 string
```

#### Get By Version

##### Request

###### End point
```
/structuredData/{id}/{tagType}/{versionId}
```
|Field| Description|
|-----|------------|
|id   | u8 array of length 64 as a base64 string.|
|tagType| Must be a permitted u64 value (9 - 11 or 10001 - (2^64 - 1)).|
|versionId| Version ID for which the Structured Data has to be fetched.|

###### Method
```
GET
```
##### Response

###### Header
```
status: 200 Ok
sd-version: {version-reference}
```

###### Body
JSON as base64 string
```javascript
Data held by the Structured Data as a base64 string
```


#### Update

Structured Data can be updated by passing the `Id, tagType and sd-version` corresponding
to the Structured Data.

For example, Say two users using an application request Structured Data with the ID ABC,
type tag 9. Assuming both the users get the same `sd-version as 5`, which means both have the same copy of the Structured Data. One user updates the Structured Data a few times and the `sd-version`
is now at `8`.
When the other user who still has `sd-version 5` - when he tries to update -the API must be able to throw a proper status code describing the conflict in sd-version (409). Based on which the applications can get the latest Structured Data and update the same again. If the `sd-version` is
not specified in the request, the latest Structured Data will be updated with the data passed in the update, which may lead to a loss of modifications which might have happened in the mean time.

##### Request

###### End point
```
/structuredData/{id}/{tagType}/{sd-version}?isVersioned=false&isPrivate=false
```
|Field| Description|
|-----|------------|
|id   | u8 array of length 64 as a base64 string.|
|tagType| Must be a permitted u64 value (9 - 11 or 10001 - (2^64 - 1)).|
|sd-version| Optional value - Checks whether the latest version of Structured Data is being updated, if the value is set to true, otherwise it will overwrite with the latest data.|


###### Method
```
PUT
```

###### Body
```
Data to be saved in the Structured Data as base64 String
```

##### Response

###### Header
```
Status: 200 Ok
```

#### Delete

##### Request

###### End point
```
/structuredData/{id}/{tagType}
```

###### Method
```
DELETE
```
##### Response

###### Header
```
Status: 200 Ok
```

### Raw Data

Raw data can be saved in the network as Immutable Data through the self-encryption process.
When raw data is written to the network, the data is self-encrypted and split into smaller
chunks and saved as Immutable Data. The self-encryption process returns a DataMap, using which
the actual data can be retrieved. This DataMap is saved to network as raw data and an ID of the
Immutable Data is obtained which refers to the DataMap.

#### Create

When the raw data is written to the network, the ID of the Immutable Data chunk referring to
the DataMap is returned.

##### Request

##### Endpoint
```
/rawData
```

##### Method
```
POST
```

##### Body
```
data as base64 string
```

#### Response

##### Header
```
status: 200 Ok
```

##### Body
```
ID [u8;64] as bas64 string
```

#### Update

##### Request

###### Endpoint
```
/rawData/{id}?offset=0
```
|Field|Description|
|-----|-----------|
|id| ID referring to the DataMap, obtained after the create operation.|
|offset| Optional parameter - if offset is not specified the data is appended to the end of the DataMap.|

###### Method
```
PUT
```

###### Body
```javascript
Data as base64 string
```

#### Response

##### Header
```
status: 200 Ok
```

#### Get

##### Request

###### Endpoint
```
/rawData/{id}?offset=0&length=100
```
|Field|Description|
|-----|-----------|
|id| ID obtained after the create operation.|
|offset| Optional parameter - if offset is not specified the data is appended to the end of the DataMap.|
|length| Optional parameter - if length is not specified the value defaults to the full length.|

###### Method
```
GET
```

#### Response

##### Header
```
status: 200 Ok
```

##### Body
```
Data as base64 String
```

# Drawbacks

1. Large file sizes cannot be supported. Streaming API will be needed for
supporting large file content.
2. Raw data cannot be completely re-written. It can only be updated (partial update)
or appended. A workaround will be to create a new DataMap.
3. Multi Signature support is not exposed in the API.

# Alternatives

None

# Unresolved Questions

When Structured Data is saved in the network directly using the low level API,
the Launcher will not be able keep an account of the Structured Data saved by the user.
Here the application will only be able to fetch the data from the network and the
users will be able to manage the data only through the application and not via the Launcher.
Should the Launcher keep track of the Structure Data and Immutable Data created by the
user?