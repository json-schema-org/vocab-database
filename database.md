# Proposal for a JSON Schema vocabulary for common database use cases.

## Change history

| Date | Author | Comments |
| --- | --- | --- |
| October 28, 2021 | srikrishnan.s.suresh@oracle.com | Add column describe, validation report use cases |
| October 13, 2021 | srikrishnan.s.suresh@oracle.com | Added code blocks |
| October 12, 2021 | beda.hammerschmidt@oracle.com | Inititial version |


## Introduction
The following write-up describes common database use cases that could benefit from JSON schema. It also proposes some extensions to JSON schema (additional keywords that can be ignored by validators not dealing with database use cases). The goal of this project is to solve common use cases in a product and vendor independent manner. Feedback, additions and modifications are appreciated. The current draft is written from a SQL-ish point of view (Oracle) but we should make sure that noSql products like MongoDB are not excluded. Also, any solution should allow vendor-specific extension. 

## Description of current state of affairs and high-level goals:

1) JSON is supported in most databases (SQL/ noSQL, open source/commercial)
2) JSON is schema flexible but users request means to 'ensure some data properties'
    - Examples: required fields, optional fields, data types, value constraints, etc.
    - JSON Schema looks like the right solution
3) SQL extensions for JSON have been added to the ISO SQL 2016 standard (SQL/JSON)
    - Oracle targets to extend the existing and standardize `'IS JSON'` SQL operator to allow schema validation. The extension will be added to the standard - making the ISON standard refer to JSON Schema (ideally an RFC is published by then).
    - For relational users it may be easier to define JSON Schema is a more sql-ish way. We therefore propose a SQL-like syntax of defining a (subset of a) JSON schema for IS JSON.
4) For efficiency, databases store JSON in non-textual form but instead in a binary encoding (Oracle OSON, Mongo BSON, Postgres JsonB,…). These binary formats also have a richer type system including types like `DATE` or `TIMESTAMP` which have no counterpart in the very simple JSON type system. We aim to allow validation of these types - this requires to extend JSON schema with an extended type system. JSON Schema already does some extensions but in a somewhat inconsistent manner: it added `'integer'` to types and `'date-time'` to format. As we want to keep existing validators working, we propose a new keyword `'sqlType'` that holds additional type instead of adding new values to the `'type'` keyword - but we're open to discuss this.
5) Some databases allow the creation of JSON data from relational tables which themselves have data types and constraints. A user that receives generated data is likely interested in a description of the data. We propose to add functionalities to auto-generate JSON schemas for tables, objects and views in a way that relational constraints and type information is preserved.
 
## JSON schema database use cases

### Validation using IS JSON

The existing and ISO standard SQL operator `'IS JSON'` performs JSON syntax checks and Oracle's implementation allows for extra conditions to be met - like unique key names. `IS JSON` can be used in the `WHERE` clause to serve as a row filter or in a check constraint to reject all insert/updates that do not satisfy the `IS JSON` condition(s).

Example: 
```
SELECT  data
FROM    mytable
WHERE   data IS JSON (WITH UNIQUE KEYS);
```

Proposal: We propose to extend 'IS JSON' to also perform a JSON Schema validation:
```
SELECT  data
FROM    mytable 
WHERE   data IS JSON (VALIDATE USING '<JSON_schema_expr>');
```
Note: `IS JSON` returns a `yes|no` answer. In the case of 'no' there is no reason given why the data failed the schema validation. To return proper error messages additional functionality is needed, like a package function, etc. This is not covered in this proposal.

`IS JSON` as shown in the example above, accepts the JSON schema using an expression - this could be a string literal, a variable or something else that contains the schema. This aspect is also not covered in this proposal. The SQL standard likely will address this.


Specific database challenge: how do we allow the validation of extended (SQL) types?

JSON has limited and fixed types: `object`, `array`, `string`, `number`, `boolean`, `null` whereas any 'real' (database) application uses temporal types like `DATE` and `TIMESTAMP`. Also, JSON does not define the range of numeric values while there is a multitude of different numeric data types for different purposes (currency, scientific data). The numeric type `DOUBLE` cannot even be mapped to a JSON number (without losses, because of the values `NAN`, `+INF`, `-INF`). 

JSON Schema has identified some shortcomings but addresses them in different (inconsistent?) ways:
 - It added `'integer'` to the type keyword,
 - It added a format keyword to ensure that a string follows ISO 8601 date-time shape.

Databases have richer type system and SQL standardized it. SQL databases like Postgre, Oracle or MySQL therefore share a more or less common type system. The internal byte representation of a typed value may be different but the semantics and value ranges are similar or the same.

### Proposal

The existing `'type'` schema keyword remains as is. Existing validators continue to work as is. A new keyword/attribute `'sqlType'` is being added. It is supported by in-database validators that the vendors create for their product (understanding their byte representation for a given type). The valid values for the `'sqlType'` keyword are the type names as defined in the SQL standard (e.g. `DATE`, `DOUBLE`, `TIMESTAM`P). Vendors can also add custom type names that only apply for their products.

Note: instead of `'sqlType'` we discussed to call it `'extendedType'` but as the values are SQL type names we favored  'sqlType'

It is very important to understand that each 'sqlType' value corresponds to exactly one `'type'` value! The correspondence becomes clear by the fact that every binary JSON representation (e.g. `BSON`, `OSON`, `JsonB`) can be converted/serialized/stringify'd to a textual representation. For example, a binary `DATE` value would become a JSON string (in `ISO 8601` format). The standard SQL operator `JSON_Serialize()` performs the conversion to textual JSON.

This 'type duality' allows us to perform validation with external validators (outside the database) as well as internal validators (inside the database) using the same schema! The external validator uses the 'type' and 'format' attributes whereas the internal validator uses the `'sqlType'` attribute. Compilation of the JSON schema needs to make sure that a compatible combination of `'sqlType'` and `'type'` is used (`DATE` cannot be matched to `BOOLEAN`). 

Examples:
```
{
  "type": "string",
  "sqlType”: "timestamp",
  "format”: "date-time"
}
```
```
{
  "type”: "string”,
  "sqlType”: "varchar(100)”,
  "maxLength”: 100
}
```
```
{
  "sqlType”: "double”,
  "oneOf" : [
   {"type": "number"},
    {"type": "string", "enum": ["NaN", "Inf", "-Inf"]}
  ]
}
```

## Automatic type coercion during insert

For databases that store JSON in a binary format the data can often be encoded into that format on the client - for example, a MongoDB driver sends BSON data to the MongoDB server. In such case the database does not have to convert textual JSON to its binary representation and hence validation using `'sqlType'` can be performed. It is to be noted that in this case no external validator is being used.
It is a different story if textual JSON is sent to the database and there into the binary representation. In this case, an external validator may accept that a value is a 'date-time' formatted string (`'type'` and `'format'` attributes) but the internal validator would reject the insertion as the same string is not a DATE `'sqlType'`. What we need is the inverse of `JSON_Serialize()`: where `JSON_Serialize()` mapped a richer type system to a small one we now need additional information to convert a 'reduced' type to its 'real' type. This information exists in the JSON schema - instead of a validation we also do a type coercion lookup. This is optional and explicit syntax is needed to enable it.

```
CREATE TABLE t1 (
  id NUMBER,
  data JSON VALIDATE COERCE USING '<json_schema_expr>'
);
```

Note: eJSON is a method where textual JSON conveys additional type information by wrapping a value into a type information object. For eJSON we do not need to perform type coercion.

Example:  `{"birthday":{"$date":"01.04.2021"}}`


*SQL-Syntax to define a JSON Schema*
(this is more a thought at this point and not a proposal yet).
The idea is to provide a syntax to define a JSON Schema which looks more like SQL. We feel such syntax would be preferred by SQL-centric developers.

Examples:
```
CREATE TABLE customers (
  data JSON {
     id       NUMBER NOT NULL, 
     name     VARCHAR2(20),
     addresses [
       { 
         street   VARCHAR2(100), 
         zip      VARCHAR2(10)
       } +
     ]
  }
)
```

is equivalent to:
```
CREATE TABLE customers (
  data JSON,
  check data IS JSON (VALIDATE USING '
  {
    "type": "object", 
    "properties": {
	"id": {"type": "number"},
	"name": {"type": "string", "maxLength": 20},
	"addresses": {
	  "type": "array",
	  "minItems":1,
	  "items": {
		"type": "object",
		"properties": {
		  "street": {"type": "string", "maxLength": 100},
		  "zip": {"type": "string", "maxLength":10}
	   }}}},
	"required": ["id"]
  }'
);
```


## Using JSON schema to describe database objects and generated JSON

This use case is not about persistent JSON data being stored in a database but about transient data being generated by the database - and then shipped to client applications.

A consumer of this JSON data likely wants to understand mor about the data: the used field names, the value types and ranges and the shape of objects/arrays. JSON schema is capable of conveying all this information - in addition it can be used to perform client-side (external) validations after data has been modified. For example, an update could be rejected because a value exceeded its maximum length.

We propose a mapping mechanism between database (column) constraint and JSON schema rules. This mapping mechanism can then be applied to describe entire tables, objects and even query results. The mapping will preserve the `'sqlType'` name, it will try to map column constraints (e.g. `NOT NULL`) to equivalent validation rules (`'required'`). Additional keywords like `'sqlDomainName'` are possible.


### Examples 
The function to invoke this mapping is not part of this proposal. We use methods of the `dbms_json` package.

#### Describe a database table

```
SELECT dbms_json.describe_schema('EMPLOYEES', 'HR') FROM dual;
{ 
  "title" : "EMPLOYEES", 
  "description" : "employees table. Contains 107 rows. References with departments,jobs, job_history 
                   tables. Contains a self reference.", 
  "type" : "object", 
  "sqlObjectType" : "table", 
  "properties" : { 
    "PHONE_NUMBER" : { 
       "description" : "Phone number of the employee; includes country code and area code",
       "type" : "string", 
       "sqlType" : "varchar2", 
       "maxLength" : 20,
    }, 
    "JOB_ID" : { 
       "description" : "Current job of the employee; foreign key to job_id column of the jobs table. 
                        A not null column.", 
       "type" : "string", 
       "sqlType" : "varchar2", 
       "maxLength" : 10
    }, 
…
```

#### Describe a database object type

```
SELECT  dbms_json.describe_schema('ADDRESS_TYP') AS SCHEMA FROM dual;
{
  "title" : "ADDRESS_TYP",
  "type" : "object",
  "sqlObjectType" : "type",
  "properties" :
  {
    "STREET" :
    {
      "type" : "string",
      "sqlType" : "varchar",
      "maxLength" : 30
    },
    "CITY" :
    {
      "type" : "string",
      "sqlType" : "varchar",
      "maxLength" : 20
    },
    "STATE" :
    {
      "type" : "string",
      "sqlType" : "char",
      "minLength" : 2,
      "maxLength" : 2
…
```
The first argument for this function is the object to be described. The second one is schema name and is optional.
`describe_schema()` function also takes in column name as optional argument. If a column name is passed,  the schema for that column alone is generated.

#### Describe a column in table/view
It is possible to restrict the JSON schema to be generated to a single column, by specifying the column name (optional parameter) in call to `describe_schema()`.

```
SELECT dbms_json.describe_schema('EMPLOYEES', 'HR', 'JOB_ID') FROM dual;

{ 
   "title": "JOB_ID",
   "description" : "Current job of the employee; foreign key to job_id column of the jobs table. 
                    A not null column.", 
   "type" : "string", 
   "sqlType" : "varchar2", 
   "maxLength" : 10
}
```
### Generating validation report
We propose a function to generate the schema validation report. The report contains information on the status of validation, i.e. if validation succeeded or failed, and the reasons for failure. Here are some examples:

(a) When validation succeeds
```
SELECT dbms_json.schema_validation_report('{"a" : 1}', '{"type": "object"}') AS report FROM dual;

REPORT
--------------------
{
  "valid" : true,
  "errors" :
  [
  ]
}
```

(b) When validation fails
```
SELECT dbms_json.schema_validation_report('{"a" : 1}', '{"type": "array"}') AS report FROM dual;

REPORT
--------------------------------------------------------------------------------
{
  "valid" : false,
  "errors" :
  [
    {
      "schemaPath" : "$",
      "instancePath" : "$",
      "code" : "JZN-00501",
      "error" : "JSON schema validation failed"
    },
    {
      "schemaPath" : "$.type",
      "instancePath" : "$",
      "code" : "JZN-00503",
      "error" : "invalid type found, actual: object, expected: array"
    }
  ]
}
```
