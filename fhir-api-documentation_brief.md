#Innovation Lab FHIR Integration Manual
##Introduction
Our back end implements a subset of the HL7 FHIR protocol, which is an HTTP-based RESTful API.  The FHIR protocol can be described as a set of object types called resources on which CRUD-operations can be performed. In this doccument, certain non-mandatory and non-relevant fields have been removed from the resource descriptions. Paths in Json objects are written in [JsonPath-format](http://goessner.net/articles/JsonPath/). For a more complete reference of FHIR, see [here](http://www.hl7.org/fhir/index.html).

FHIR supports five different basic operations that can be performed on resources.

* Read - Retrieve a single resource from the system. Implemented by HTTP GET.
* Search - Retrieve a list of a specific resource type based on a set of optional filter parameters. Implemented by HTTP GET.
* Create - Insert a single resource. Implemented by HTTP POST.
* Update - Modify an existing resource. Effectively replacing a resource with a new one but with the same logical resource id. Implemented by HTTP PUT.
* Delete - Delete a single resource from the system. Implemented by HTTP DELETE.

Although all operations are supported by all resources mentioned here, this document only describes those that are relevant to this particular integration. It will most likely be useful for testing purpouses to modify the existing data on the test server through the API.

## Data format
FHIR supports either Json or XML for data-representation. All examples in this documents will use the Json format. The data format for the response is determined by an HTTP parameter named `_format` which can be appended to any request. Valid values are `json` and `xml`. 

##Authorization
Interacting with the FHIR-resource server requires an API-key in each request. This is communicated using the basic auth method which uses the `Authorization` HTTP Header.

This is the format of the Authorization header:
```
Authorization: Basic [auth string]
```
Where `auth string` is derrived from "user:API_KEY" in which API_KEY should be replaced by the actual API key. The string is then encoded using base-64. No quotes should be included.

##FHIR Resources

###Observation
The Observation resource represents a single vital sign measurement by a medical device.

Relevant fields in this resouce are 

* `$.code.coding[*].code` - A code representing which vital sign was measured.
  * `MDC_PULS_OXIM_PULS_RATE` - Patient's heart beat rate.
  * `MDC_PULS_OXIM_SAT_O2` - Blood oxygen saturation percentage.
  * `MDC_PRESS_BLD_NONINV_DIA` - Diastolic blood pressure.
  * `MDC_PRESS_BLD_NONINV_SYS` - Systolic blood pressure.
  * `MDC_PRESS_BLD_NONINV_MEAN` - Mean blood pressure.
* `$.subject.reference` - This is a reference to the patient resource containing personal information for whom the value was measured.
* `$.effectiveDateTime` - When the measurement was taken.
* `$.valueQuantity.value` - The measured value. 
* `$.valueQuantity.unit` - Textual descriptions of the unit in use.

#### Search

Observation objects can be searched with the following filter parameters which are added as query-parameters in a HTTP GET request. The response is a Bundle object containing a list of the resulting Observation-resources. The resources are found under `$.entry[*].resource`.

The following search parameters are applicable:
* `subject` - The personal identification number of the patient, e.g. `subject=19121212-1212`. 
* `date` - Refers to the `start` field. Should be a ISO 8601-formatted date-time value, optionally prefixed with an inequality operator. E. g. `date=>=2015-02-07T12:12:12Z`. This parameter can also be repeated twice with upper and lower limits to specify a range.
* `code` - Refers to a code which vital sign was measured. Valid values are the same as for `$.code.coding[*].code`.

##### Example
This example retrieves the latest pulse rate measurement for a given patient.
Search request:
```
GET /fhir/Observation?code=MDC_PULS_OXIM_PULS_RATE?date=>=2016-02-19T14:00:00Z&date=<=2016-02-19T14:00:00Z&subject=19121212-1212
Authorization: Basic [auth string]
```

Response body:
```javascript
{
  "resourceType": "Bundle",
  "entry": [
    {
    "resource": {
        "resourceType": "Observation",
        "id": "667810cf-ac01-4b11-996f-14ba817b7445",
        "code": {
          "coding": [
            {
              "system": "MDC",
              "code": "MDC_PULS_OXIM_PULS_RATE"
            }
          ],
          "text": "Pulse Rate from Plethysmogram {Philips} Pulse Rate {Draeger} Pulse Oximetry Peripheral Heart Rate {GE} Pulse Rate {GE} Pulse rate (from pulse oximeter) {VIASYS} Pulse Rate {Nuvon}"
        },
        "subject": {
          "reference": "Patient/191212-1212"
        },
        "effectiveDateTime": "2016-03-10T08:00:00Z",
        "performer": [
          {
            "reference": "Device/C1007-123"
          }
        ],
        "valueQuantity": {
          "value": 83,
          "unit": "/min 1/min {count}/min /min 1/min {beat}/min {beats}/min /min 1/min {pulse}/min {pulses}/min"
        }
      }
    }
  ]
}

```
#References
* HL7 FHIR http://www.hl7.org/fhir/
