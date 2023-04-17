# STAC API - Aggregation Extension Specification

- **Title:** Aggregation
- **OpenAPI specification:** TBD
- **Conformance Classes:**
  - <https://api.stacspec.org/v0.2.0/aggregation>
- **Scope:** STAC API - Core
- **[Extension Maturity Classification](https://github.com/radiantearth/stac-api-spec/tree/main/README.md#maturity-classification):** Proposal
- **Dependencies:**
  - [STAC API - Core](https://github.com/radiantearth/stac-api-spec/tree/v1.0.0-rc.2/core)
- **Owner**: @philvarner

The purpose of the Aggregation Extension is to provide an endpoint similar to the Search endpoint
(`/search`), but which will provide aggregated information on matching Items rather than the
Items themselves. This is highly influenced by the Elasticsearch and OpenSearch aggregation
endpoint, but with a more regular structure for responses.

## STAC Endpoints

| Endpoint                       | Returns                                                                                                        | Description                                                                                          |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `/aggregate`                   | AggregationCollection                                                                                          | Retrieves an aggregation of the group of Items matching the provided predicates                      |
| `/aggregations`                | JSON                                                                                                           | Retrieves a schema defining the terms which are available for aggregation over.                      |
| `/{collectionId}/aggregate`    | Retrieves an aggregation of the group of Items matching the provided predicates, restricted to this collection |
| `/{collectionId}/aggregations` | JSON                                                                                                           | Retrieves a schema defining the terms which are available for aggregation over only this collection. |

The `/aggregate` endpoint behaves similarly to the `/search` endpoint, but instead of returning an
ItemCollection of Items, it instead returns aggregated information over the same matching Items
in the form of an **AggregationCollection** of **Aggregation** entities.

The `/aggregations` endpoint returns a schema-like document describing aggregations that can be requested,
similarly to how the queryables endpoint is used in the Filter Extension. These can be passed in
the `aggregations` parameter to `/aggregate` to restrict the aggregations performed.

Other STAC API Extensions, notably Query and Filter, can be advertised
with <https://api.stacspec.org/v0.2.0/aggregation#query> and
<https://api.stacspec.org/v0.2.0/aggregation#filter>.

For the links in both landing page (`/`) and collection (`/collections/{collectionId}`), it is
**required** to add a link with the `rel` value set to `aggregate` that refers to the relevant
aggregate endpoint in the `href` property, and a link with the `rel` value set to `aggregations`
that refers to the relevant aggregations endpoint in the `href` property. The `method` field can
be defined with `GET` or `POST` to advertise support for those HTTP methods, or if no method field
is defined, only `GET` is supported.

## Filter Parameters and Fields

The filters for `/aggregate` are the same as those for `/search` that are semantically meaningful
(e.g., `limit` has no meaning when doing aggregations). These filters are passed as query string
parameters or JSON entity fields.  For filters that represent a set of values, query parameters
should use comma-separated
string values and JSON entity attributes should use JSON Arrays.

| Parameter    | Type             | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ------------ | ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| datetime     | string           | Single date+time, or a range ('/' seperator), formatted to [RFC 3339, section 5.6](https://tools.ietf.org/html/rfc3339#section-5.6). Use double dots `..` for open date ranges.                                                                                                                                                                                                                                                                                                                                                                                                    |
| bbox         | \[number]        | Requested bounding box.  Represented using either 2D or 3D geometries. The length of the array must be 2*n where n is the number of dimensions. The array contains all axes of the southwesterly most extent followed by all axes of the northeasterly most extent specified in Longitude/Latitude or Longitude/Latitude/Elevation based on [WGS 84](http://www.opengis.net/def/crs/OGC/1.3/CRS84). When using 3D geometries, the elevation of the southwesterly most extent is the minimum elevation in meters and the elevation of the northeasterly most extent is the maximum. |
| intersects   | GeoJSON Geometry | Searches items by performing intersection between their geometry and provided GeoJSON geometry.  All GeoJSON geometry types must be supported.                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ids          | \[string]        | Array of Item ids to return. All other filter parameters that further restrict the number of search results (except `next` and `limit`) are ignored                                                                                                                                                                                                                                                                                                                                                                                                                                |
| collections  | \[string]        | Array of Collection IDs to include in the search for items. Only Items in one of the provided Collections will be searched                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| aggregations | \[string]        | A list of aggregations to compute and return                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |

Only one of either **intersects** or **bbox** should be specified.  If both are specified, a 400 Bad Request response should be returned.

**aggregations**: There are no named aggregations that must be implemented. All aggregations which
are available should be advertised in the root `rel="aggregations"` link. By default, the
`/aggregations` endpoint must only return the named aggregations specifically requested via
the `aggregations` parameter.

For the collection subresource, only collections that are specifically advertised by that collection's
`/aggregations` endpoint should be allowed, and if any others are requested, a 400 Bad Request
exception should be returned. At the root-level `/aggregate` endpoint, any aggregation may be
requested, even if it does not apply to all items in the result set.

It is recommended to suffix count aggregations with `_count` and frequency distribution aggregations
with `_frequency`.

This is a list of recommended aggregations to implement:

- total_count (Single Value of type integer)
- datetime_min (Single Value of datetime)
- datetime_max (Single Value of datetime)
- collection_frequency (Term Count)
- cloud_cover_frequency (Discrete Range)
- datetime_frequency (Datetime Range, automatic interval detection) -- detect a reasonable
  interval based on the datetime range and distribution of data? Implementation specific.

## AggregationCollection fields

This object describes a STAC AggregationCollection, which is the analog of an ItemCollection for the `/aggregate` operation.

| Field Name   | Type           | Description                                          |
| ------------ | -------------- | ---------------------------------------------------- |
| type         | string         | **REQUIRED** Always "AggregationCollection".         |
| aggregations | \[Aggregation] | **REQUIRED** A possibly-empty array of Aggregations. |

## Aggregation fields

| Field Name | Type                         | Description                                                                                         |
| ---------- | ---------------------------- | --------------------------------------------------------------------------------------------------- |
| name       | string                       | **REQUIRED** The unique indentifier of the aggregation.                                             |
| data_type  | string                       | **REQUIRED** The data type of the aggregation                                                       |
| buckets    | \[Bucket]                    | If the aggregation bucketizes Items, they are defined here.                                         |
| overflow   | integer                      | The count of Items that were not categorized into any of the buckets defined by the `buckets` field |
| value      | string or number or datetime | For a Single Value aggregation, a JSON-type representation of the result value.                     |

One of either **buckets** or **value** is required.

**name** An identifier for the aggregation result. Should be identical to the value passed to the `aggregations` query parameter.

**data_type** numeric, string, datetime, interval_year, interval_month, interval_week
interval_day, interval_hour, interval_minute, interval_second

**buckets** If the aggregation is a Term Count, Datetime Range, or Discrete Range, these are
the "buckets" into which each matching Item is categorized.

**overflow** Some implemenation data stores may have limitations on the aggregation queries
that can be performed on them. For example, Elasticsearch limits the number of buckets for
a query to 10,000 for performance reasons.  Overflow indicates that there were Items matched
by the query that are not accounted for in the count of any of the response buckets.

**value** For Single Value aggregations, this is a representation of the result value as the
equivalent JSON type. If the type of the value being aggregated over is a datetime, this is
an RFC 3339 datetime, e.g., "2020-08-12T19:06:09Z".

## Bucket fields

| Field Name | Type    | Aggregation Types | Description |
| ---------- | ------- | ----------------- | ----------- |
| key        | string  | all               |             |
| data_type  | string  | all               |             |
| frequency  | integer | all               |             |
| from       | numeric | all               |             |
| to         | numeric | all               |             |

## Aggregation Types (TBD)

This section has not been updated to match the previous definitions.

### Single Value Aggregation

effectively a single Term Count Bucket lifted up one level

**todo** (diff for String, Numeric, and Datetime)

Example:

```json
    {
        "type": "AggregationCollection",
        "aggregations": [
            {
                "key": "datetime_min",
                "value": "2000-02-16T00:00:00.000Z",
                "value_as_type": 1.506592E+11
            }
        ]
    }
```

### Term Count Aggregation

- enumeration count multi bucket one per unique value

Example:

```json
    {
        "type": "AggregationCollection",
        "aggregations": [
            {
                "key": "collections",
                "buckets": [
                    { 
                        "key": "sentinel2_l1c",
                        "value": "12649072",
                        "value_as_type": 12649072
                    },
                    {
                        "key": "landsat8_l1tp",
                        "value" : "1071997",
                        "value_as_type": 1071997
                    }
                ],
                "overflow": 23414
            }
        ]
    }
```

### Discrete Range Aggregation

Fields:

- key (string)
- key_as_type ()
- from (optional, missing indicates an open interval) inclusive
- to (optional, missing indicates an open interval) exclusive
- value (integer)

Example:

```json
    {   
        "type": "AggregationCollection",
        "aggregations": [
            {
                "key": "cloud_cover",
                "buckets": [
                    { 
                        "key": "*-5.0",
                        "to": 5,
                        "value" : "8644819",
                        "value_as_type" : 8644819
                    },
                    {
                        "key": "5.0-10.0",
                        "from": 5,
                        "to": 10,
                        "value" : "5644819",
                        "value_as_type" : 5644819
                    },
                    {
                        "key": "10.0-*",
                        "from": 10,
                        "value" : "7644819",
                        "value_as_type" : 7644819
                    }
                ]
            }
        ]
    }
```

### Datetime Range Aggregation

Fields:
datetimes are RFC 3339 string values

- key (string)
- key_as_type (datetime in milliseconds?)
- value (integer) (ES: doc_count)

Example:

```json
    {   
        "type": "AggregationCollection",
        "aggregations": [
            {
                "key": "datetime_yearly",
                "buckets": [
                    { 
                        "key": "2000-01-01T00:00:00.000Z",
                        "key_as_type": 946684800000,
                        "to": 5,
                        "value" : "8644819",
                        "value_as_type" : 8644819
                    },
                    {
                        "key": "2001-01-01T00:00:00.000Z,
                        "key_as_type": 978307200000,
                        "from": 5,
                        "to": 10,
                        "value" : "5644819",
                        "value_as_type" : 5644819
                    },
                    {
                        "key": "2002-01-01T00:00:00.000Z,
                        "key_as_type": 1009843200000,
                        "from": 10,
                        "value" : "7644819",
                        "value_as_type" : 7644819
                    }
                ],
                "interval": "year",
                "overflow": 98373 
            }
        ]
    }
```
