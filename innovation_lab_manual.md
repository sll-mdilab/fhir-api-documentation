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
###Appointment
#### Search
Appointment objects can be searched with the following filter parameters which are added as query-parameters in a HTTP GET request. The response is a Bundle object containing a list of the resulting Appointment-resources. The resources are found under `$.entry[*].resource`. Both the subject patient and the medical personel performing the procedure is listed under `$.entry[*].resource.participant` for which the types `SBJ` and `PPRF` are used respectively.

* `date` - Refers to the `start` field. Should be a ISO 8601-formatted date-time value, optionally prefixed with an inequality operator. E. g. `date=>=2015-02-07T12:12:12Z`. This parameter can also be included twice with upper and lower limits to specify a range.
* `status` - Refers to the `status` field. Should be a string value. E. g. `status=booked`. If an appointment is cancelled by the patient through his/her questionnaire answer, the `status` field should be set to `cancelled`.

```
GET /fhir/Appointment[?parameter=value][&parameter=value][&parameter=value]...
```

##### Example
Example search request:
```
GET /fhir/Appointment?date=>=2015-02-04T14:00:00Z&date=<=2015-02-04T14:00:00Z&status=booked&_format=json
Authorization: Basic [auth string]
```

Example response body:
```javascript

{
  "resourceType": "Bundle",
  "id": "68c464c2-a5f0-44b7-ac39-1acf54ef835c",
  "type": "searchset",
  "total": 1,
  "entry": [
    {
      "resource": {
        "resourceType": "Appointment",
        "id": "dc689f19-7179-4101-8d23-7f62814901eb",
        "status": "booked",
        "extension" : [
          {
            "url":"http://sll-mdilab.net/fhir/Appointment#processedByAgent",
            "valueBoolean" : true
          }
        ],
        "type": {
          "coding": [
            {
              "code": "408466002",
              "system": "http://hl7.org/fhir/ValueSet/c80-practice-codes",
              "display": "Cardiac surgery"
            }
          ]
        },
        "description": "Double-bypass heart surgery.",
        "start": "2016-03-10T09:00:00Z",
        "end": "2016-03-10T12:00:00Z",
        "comment": "We are going to perform the Double-bypass heart surgery as discussed earlier.",
        "participant": [
          {
            "actor": {
              "reference": "Patient/19121212-1212",
              "display": "Tolvan Tolvansson"
            },
            "type": { 
              "coding": [
                {
                  "code": "SBJ",
                  "system": "http://hl7.org/fhir/ValueSet/v3-ParticipationType"
                }
              ]
            }
          },
          {
            "actor": {
              "reference": "Practitioner/19131313-1313",
              "display": "Adam Bertilsson"
            },
            "type": { 
              "coding": [
                {
                  "code": "PPRF",
                  "system": "http://hl7.org/fhir/ValueSet/v3-ParticipationType"
                }
              ]
            }
          }
        ]
      }
    }
  ]
}
```


###Patient
The Patient resource represents the personal details of a single patient. Relevant in this case is mainly `$.name` and `$.telecom[?(@.system="email")].value` (for email).

#### Read
Patients are retrieved using HTTP GET with a supplied resource id. 
```
GET /fhir/Patient/[id]
```
##### Example
Example read request:
```
GET /fhir/Patient/19121212-1212?_format=json
Authorization: Basic [auth string]
```

Example response body:
```
{
  "resourceType": "Patient",
  "id": "191212-1212",
  "active": true,
  "name": [
    {
      "use": "official",
      "family": [
        "Tolvansson"
      ],
      "given": [
        "Tolvan"
      ]
    }
  ],
  "telecom": [
    {
      "system": "email",
      "value": "tolvan.tolvansson@karolinska.se"
    }
  ],
  "gender": "male",
  "birthDate": "1912-12-12"
}
```

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
* `_count` - Imposes a limit on then number of results. E. g. `_count=3`.
* `_sort` - Sort results based on a chosen search parameter (used in the request or not). Supports prefixes `:asc` and `:desc` (:asc is default). E. g. `sort:desc=date`

##### Example
This example retrieves the latest pulse rate measurement for a given patient.
Search request:
```
GET /fhir/Observation?code=MDC_PULS_OXIM_PULS_RATE&subject=19121212-1212date=>=2016-03-10T07:00:00Z&date=<=2016-03-10T09:00:00Z&_count=1&_sort:desc=date
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
          "reference": "Patient/19121212-1212"
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

###QuestionnaireResponse
The QuestionnaireResponse represents a single set of answers submitted by a specific patient and/or practitioner. This Resouce contains a hierarchy of `group` and `question`-objects. A group with `linkId` equal to `pab` makes up the root. Under the root, there are one or two groups for each question. The first group contain answers supplied by the patient, the second contain validatin and comments made by the practitioner (these are added by an Update request).

When the questionnaire is filled out by the patient, the resource should be created (using HTTP POST) with `$.status` set to `in-progress`. When it's then validated and commented by a medical practitioner, that same object should be updated (using HTTP PUT) with `$.status` set to `completed`.

#### Create
QuestionnaireResponse resource objects are created using HTTP POST.
```
POST /fhir/QuestionnaireResponse
Content-Type: application/json+fhir
Authorization: Basic [auth string]
```
##### Example

Request body:
```javascript
{
  "resourceType": "QuestionnaireResponse",
  "status": "in-progress",
  "subject": {
    "reference": "Patient/191212-1212"
  },
  "authored": "2016-02-04T14:15:00Z",
  "extension" : [
    {
      "url":"http://sll-mdilab.net/fhir/QuestionnaireResponse#appointment",
      "valueReference" : {
        "reference" : "Appointment/dc689f19-7179-4101-8d23-7f62814901eb"
      }
    }
  ],
  "group": {
    "linkId": "pab",
    "title": "Preanestesibedömning",
    "group": [
      {
        "linkId": "komplikationerSlakting",
        "group": [
          {
            "linkId": "komplikationerSlaktingPatient",
            "question": [
              {
                "linkId": "komplikationerSlaktingExisterar",
                "text": "Har någon släkting haft komplikationer i samband med operation?",
                "answer": [
                  {
                    "valueBoolean": true
                  }
                ]
              },
              {
                "linkId": "komplikationerSlaktingBeskrivning",
                "answer": [
                  {
                    "valueString": "Min far fick en komplikation i samband med en blindtarmsoperation för 3 år sedan."
                  }
                ]
              }
            ]
          }
        ]
      }
    ]
  }
}
```

#### Update
QuestionnareResponse resource objects are updated (modified) using HTTP PUT with a supplied resource id. 
```
PUT /fhir/QuestionnareResponse/[id]
```
##### Example

```
PUT /fhir/QuestionnareResponse/553dc4ad-f113-49b8-9aba-d3930c946082
Content-Type: application/json+fhir
Authorization: Basic [auth string]
```

Request body:
```javascript
{
  "resourceType": "QuestionnaireResponse",
  "status": "completed",
  "subject": {
    "reference": "Patient/191212-1212"
  },
  "authored": "2016-03-10T08:30:00Z",
  "extension" : [
    {
      "url":"http://sll-mdilab.net/fhir/QuestionnaireResponse#appointment",
      "valueReference" : {
        "reference" : "Appointment/dc689f19-7179-4101-8d23-7f62814901eb"
      }
    }
  ],
  "group": {
    "linkId": "pab",
    "title": "Preanestesibedömning",
    "group": [
      {
        "linkId": "komplikationerSlakting",
        "group": [
          {
            "linkId": "komplikationerSlaktingPatient",
            "question": [
              {
                "linkId": "komplikationerSlaktingExisterar",
                "text": "Har någon släkting haft komplikationer i samband med operation?",
                "answer": [
                  {
                    "valueBoolean": true
                  }
                ]
              },
              {
                "linkId": "komplikationerSlaktingBeskrivning",
                "answer": [
                  {
                    "valueString": "Min far fick en komplikation i samband med en blindtarmsoperation för 3 år sedan."
                  }
                ]
              }
            ]
          },
          {
            "linkId": "komplikationerSlaktingPersonal",
            "question": [
              {
                "linkId": "komplikationerSlaktingValiderad",
                "text": "Validerad",
                "answer": [
                  {
                    "valueBoolean": true
                  }
                ]
              },
              {
                "linkId": "komplikationerSlaktingKommentar",
                "text": "Kommentar vårdpersonal:",
                "answer": [
                  {
                    "valueString": "Denna komplikation påverkar inte riskbedömningen för ingreppet."
                  }
                ]
              }
            ]
          }
        ]
      }
    ]
  }
}
```
#References
* HL7 FHIR http://www.hl7.org/fhir/
