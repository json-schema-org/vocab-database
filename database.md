# Proposal for a JSON Schema vocabulary for common database use cases.

## Introduction

The following write-up describes common database use cases that could benefit from JSON schema. It also proposes some extensions to JSON schema (additional keywords that can be ignored by validators not dealing with database use cases). The goal of this project is to solve common use cases in a product and vendor independent manner. The proposal also caters to vendors (not database ones necessarily) that process JSON using a binary format. The current draft is written from a SQL point of view (Oracle) but we should make sure that NoSQL products like MongoDB are not excluded. Also, any solution should allow vendor-specific extensions. Feedback, request for additions and modifications in this proposal are appreciated. 

## Current state of affairs and high-level goals:

1) JSON as a storage data type is supported in most database products (SQL/NoSQL, open source/commercial).
2) JSON is schema flexible but users request means to enforce some data properties on it.
    - Examples: required fields, optional fields, data types, constraints on values etc.
    - JSON Schema looks like the right solution.
3) SQL extensions for JSON have been added to the ISO SQL 2016 standard (SQL/JSON).
    - Oracle aims to extend the existing and standardized `IS JSON` SQL operator to allow schema validation of JSON data.
    - The extension will be added to the ISO standard - making the ISO standard refer to JSON Schema (ideally an RFC is published by then).
    - For relational users, it may be easier to define JSON Schema is a more SQL-ish way. We therefore also propose a SQL-like syntax for defining a (subset of a) JSON schema for `IS JSON`.
4) For efficiency, databases store JSON not as text but in a binary format (example, Oracle OSON, Mongo BSON, Postgres JsonB, etc.). These binary formats also have a richer type system, including types like `BINARY`, `FLOAT`, `DOUBLE`, `DATE`, `TIMESTAMP`, `INTERVAL`, etc. which have no counterpart in the standard JSON type system. We aim to allow validation of these extended types - this requires extending JSON schema vocabulary with an extended type system.
5) JSON Schema already offers some extensions, but in a somewhat inconsistent manner; it added `'integer'` to types, `'date-time'` and `interval` to format. As we want to keep existing validators working, we propose a new keyword `'extendedType'` that holds additional type instead of adding new values to the existing `'type'` or `'format'` keyword.
6) Some databases allow the creation of JSON data from relational tables that have columns with a data type and constraints. A user that receives generated data is likely interested in a description of the data. We therefore propose to add interfaces to generate JSON schemas for database objects such as tables, views and types in such a way that relational constraints and type information are preserved.
 
## JSON Schema - database use cases

### Validation using IS JSON

The existing and ISO standard SQL operator `IS JSON` performs JSON syntax checks and Oracle's implementation allows for extra conditions to be met - like unique key names. `IS JSON` can be used in the `WHERE` clause of a query to serve as a row filter or in a check constraint to reject any insert/update that do not satisfy the `IS JSON` condition(s).

Example: 
```
SELECT  data
FROM    mytable
WHERE   data IS JSON (WITH UNIQUE KEYS);
```

#### Proposal

We propose to extend 'IS JSON' operator to also perform JSON Schema validation. For example,
```
SELECT  <column_list>
FROM    <table_name> 
WHERE   <column_name> IS JSON VALIDATE USING '<JSON_schema_expr>';
```
Note: `IS JSON` returns a `true|false` answer. In the case of `false` return, there is no reason given for why the JSON instance failed the schema validation. To return proper error messages additional functionality is needed, like a package function, etc.


`IS JSON` expression as shown in the example above, accepts the JSON schema using an expression - this could be a schema string literal, a bind variable or an identifier (database object) from which schema can be inferred. This aspect is not covered in this proposal. The SQL standard likely will address this.

The following example illustrates the use of `IS JSON` in a `CHECK` constraint expression:

```
CREATE TABLE jtab (
  id    NUMBER(6)  PRIMARY KEY,
  jcol  JSON       CHECK (jcol IS JSON VALIDATE USING '{
	"type": "object",
	"properties": {
		"id": {
			"type": "number"
		},
		"name": {
			"type": "string",
			"maxLength": 20
		},
		"addresses": {
			"type": "array",
			"minItems": 1,
			"items": {
				"type": "object",
				"properties": {
					"street": {
						"type": "string",
						"maxLength": 100
					},
					"zip": {
						"type": "string",
						"maxLength": 10
					}
				}
			}
		}
	},
	"required": ["id"]
}')
);
```

### Validation of extended types

JSON has limited and fixed types: `object`, `array`, `string`, `number`, `boolean`, `null` whereas any real (database) application uses other types such as `BINARY`, `FLOAT`, `DOUBLE`, `DATE`, `TIMESTAMP`, `INTERVAL` etc. Also, JSON does not define the range of numeric values while there is a multitude of different numeric data types for different purposes (currency, scientific data). The numeric type `DOUBLE` cannot even be mapped to a JSON number (without losses, because of the values `"NAN"`, `"+INF"`, `"-INF"`). 

JSON Schema has identified some shortcomings but addresses them in different (inconsistent?) ways:
 - It added `'integer'` to the type keyword,
 - It added a format keyword to ensure that a string follows ISO 8601 date-time shape.

Databases have richer type system and SQL standardized it. SQL databases like Oracle, Postgres, or MySQL therefore share a more or less common type system. The internal byte representation of a typed value may differ but the semantics and value ranges are similar or the same.

#### Proposal

The behavior of existing `'type'` keyword in JSON schema remains as is, so that existing (third party) validators continue to work as is without any modifications. A new keyword/attribute `'extendedType'` is being added and supported by vendor specific validators. `'extendedType'` can be used instead of `'type'` keyword, or in conjunction as well. Incompatible values for `'type'` and `'extendedType'` will fail validation (just like JSON schema `false`), but is not an invalid schema. For instance,  `{"type": "boolean", "extendedType": "date"}` is a valid schema, but document validation will fail, as a type cannot be both boolean and date.

The following standard values are valid:
 - object
 - array
 - string
 - number
 - integer
 - boolean
 - null

The behavior of the above values for the `'extendedType'` keyword in the JSON schema is the same as that of `'type'` keyword.

The following values are extended types and must be supported by all validators that support `'extendedType'` keyword:
 - date
 - timestamp
 - timestampTz
 - interval
 - binary
 - double
 - float

Just like `'type'`, `'extendedType'` keyword can have a string or an array value. The following schema documents are valid:

`{"extendedType": "float"}`

`{"extendedType": ["date", "timestamp", "timestampTz"]}`

We define a set of values allowed for `'extendedType'` based on existing standardized type systems like SQL, SQL/JSON. `'extendedType'` values space is a super set of `'type'`'s value space, i.e. it allows all values permitted in type and in addition supports other types as well. 

Most of these types are already supported in the JSON type and the SQL operator `JSON_Serialize()` specifies how these values undergo serialization to textual JSON. Both JSON type and `JSON_Serialize()`  SQL operator are already part of the open ISO SQL standard.

`JSON_Serialize()` defines how a richer type system gets 'reduced' to a smaller type system by lossy type conversions. For example, binary date and interval values undergo conversions to strings in ISO 8601 format. A binary value will be coverted to a hexadecimal encoded string. The mapping defined by `JSON_Serialize()` therefore defines how a third-party validator treats the extended types. A vendor validator which has been built specifically for a product (e.g. Oracle Database validator) can validate the richer type system, as well as strings that are in the format defined by the mapping. For example, a validator that supports `'extendedType'` successfully  validates the document instance: `"2022-01-31"` against the schema: `{"extendedType": "date"}`. However, the document instance `"some string"` fails validation against the schema: `{"extendedType": "interval"}`, as `"some string"` is not a valid ISO 8601 interval string.

#### Keywords applicable for extended types
The keywords that are applicable for number and integer types are also applicable for the extended numeric types: `float` and `double`:
```
{
  "extendedType": ["float", "double"],
  "minimum": 35.75
  "multipleOf": 5.25
}
```

#### Extensibility
Vendors are allowed to extend the set of allowed values for `'extendedType'`. This makes the `'extendedType'` open for vendor-specific extensions - although we want to keep this list minimal. A validator that does not support this keyword will treat it as a annotation. On the other hand, a vendor that does support the keyword, can validate, and in addition, perform type coercion to a relevant binary type (if vendor supports a binary format for JSON), as discussed in the subsequent section.

### Type coercion/cast during DMLs

For databases that support a binary JSON format, data can be encoded on the client. For example, a MongoDB driver sends BSON data to the MongoDB server. In such cases, the database does not have to convert textual JSON to its binary representation and hence validation using `'extendedType'` can be performed. It is to be noted that in this case no external validator is being used at all. On the other hand, if textual JSON is sent to the database, it is followed by an encoding process to a binary representation (server-side encoding). In such circumstances, the schema validator can operate in *COERCE* or *CAST* mode. That is, the binary encoder can use the value specified for the `'extendedType'` keyword in the JSON schema and encode the scalar field to its binary representation.

This facility is optional and an explicit syntax is needed to enable it. We will use `'CAST'` keyword to specify this mode of operation:

```
CREATE TABLE jtab (
  id NUMBER,
  jcol JSON CHECK(jcol IS JSON VALIDATE CAST USING '{
    "type": "object",
    "properties": {
      "firstName": {
        "extendedType": "string",
        "maxLength": 50
      },
      "birthDate" : {
        "extendedType": "date"
      }
    },
    "required": ["firstName", "birthDate"]
  }')
);
```

The following textual JSON is a valid document per the above schema:

```
{
  "firstName": "Scott",
  "birthDate": "1990-04-02"
}
```

In *COERCE/CAST* mode, vendors can encode the `"firstName"` and `"birthDate"` fields in the textual JSON to string and binary date formats respectively.

```
{
  "type": "object"
  "salary": {
    "extendedType": "integer"
  }
}
```

Given the above schema, in CAST mode, the value for the field `"salary"` will be cast to an integer. Example:

```
{
  "salary": 88733.50
}
```
will be cast to

```
{
  "salary": 88733
}
```
by the validator. For that matter, if the salary value was a string with numeric characters, example `"88733.50"` , it would be cast to the same integer value `88733` as well.

Keywords for specifying range, can be used on `date`, `timestamp`, `timestampTz` and `interval` extended types:
```
{
  "extendedType": "date"
  "minimum": "2020-03-04"
}
```

The above schema validates date values that equal or greater than the date value `2020-03-04`.

### SQL syntax for JSON Schema
(This is more a thought at this point and not a proposal yet).

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

### Describing database objects using JSON schema

This use case is not about persistent JSON data in a database, but about transient data being generated by the database - and then shipped to client applications. Some databases allow the creation of JSON data from relational tables which themselves have data types and constraints.

A consumer of this JSON data likely wants to understand more about the data: the used field names, the value types and ranges and the shape of objects/arrays. JSON schema is capable of conveying all this information - in addition it can be used to perform client-side (external) validations after data has been modified. For example, an update could be rejected because a value exceeded its maximum length.

We propose a mapping mechanism between database (column) constraint and JSON schema rules. This mapping mechanism can then be applied to describe entire tables, objects and even query results. The mapping will preserve the `'extendedType'` name, it will try to map column constraints (e.g. `NOT NULL`, `CHECK`) to equivalent validation rules. Additional keywords like `'sqlDomainName'` are possible.

#### Keywords for describing database objects

The following keywords are used to specify the attributes of the object:
| Keyword         | Value type      | Value description |
| ---             | ---             | --- |
| sqlObjectName	  | string          | Name of the object being described |
| sqlObjectOwner	| string          | Name of the owner of the object |
| sqlObjectType	  | string          | Type of the database object |
| description	    | string          | Comment on the object (table/view) |
| sqlPrimaryKey   | string or array | Name(s) of the primary key column(s) |
| sqlForeignKey   | array           | Foreign keys on the table |

The following keywords are used to specify the attributes of the column values:
| Keyword         | Value type       | Value description |
| ---             | ---              | --- |
| extendedType	  | string or array  | Type of the JSON data generated by converting the column value to JSON |
| description	    | string           | Comments on the column |
| maxLength	      | integer          | Maximum size of the value |
| minLength	      | integer          | Minimum size of the value |
| sqlPrecision    | integer	         | Precision for numeric, timestamp types |
| sqlScale        | integer          | Scale for numeric types |

#### Foreign Key

For now, the generated JSON schema will list the table name and column names for each of the foreign keys. 

```
"sqlForeignKeys" : [
  {
    "DEPTID": {
      "sqlObjectName": "DEPARTMENTS",
      "sqlObjectOwner": "HR"
      "sqlColumnName": "DEPARTMENT_ID"
    }
  } 
]
```

The above schema means that the column `DEPTID` is a foreign key defined and it references the column `DEPARTMENT_ID` in the table `HR.DEPARTMENTS`.

Note that  a composite foreign key is also possible:

```
"sqlForeignKeys" : [
  {
    "ORDER_ID": {
      "sqlObjectName": "ORDERS",
      "sqlObjectOwner": "OE"
      "sqlColumnName": "ORDERID"
    },
    "CUSTOMER_ID": {
      "sqlObjectName": "ORDERS",
      "sqlObjectOwner": "OE"
      "sqlColumnName": "CUSTID"
    }
  }  
]
```

#### Examples 
We use the function `describe` in `dbms_json_schema` package.

##### Describe a database table

```
SELECT dbms_json_schema.describe('EMPLOYEES', 'HR') FROM dual;
{ 
  "sqlObjectName" : "EMPLOYEES", 
  "sqlObjectType" : "table", 
  "sqlObjectOwner": "HR"
  "description" : "employees table. Contains 107 rows. References with departments,jobs, job_history 
                   tables. Contains a self reference.", 
  "type" : "object", 
  "sqlPrimaryKey": "EMPLOYEE_ID",
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

##### Describe a database object type

```
SELECT  dbms_json_schema.describe('ADDRESS_TYP', 'SCOTT') AS SCHEMA FROM dual;
{
  "sqlObjectName" : "ADDRESS_TYP",
  "sqlObjectOwner" : "SCOTT",
  "sqlObjectType" : "type",
  "type" : "object",
  "properties" :
  {
    "STREET" :
    {
      "extendedType" : "string",
      "maxLength" : 30
    },
    "CITY" :
    {
      "extendedType" : "string",
      "maxLength" : 20
    },
    "STATE" :
    {
      "extendedType" : "string",
      "maxLength" : 2
…
```
The first argument for this function is the name of the object to be described. The second one is schema name and is optional.
`describe()` function also takes in column name as optional argument. If a column name is passed,  the schema for that column alone is generated. Only table, view, or type can be described, not other types.

##### Describe a column in table/view
It is possible to restrict the JSON schema to be generated to a single column, by specifying the column name (optional parameter) in call to `describe()`.

```
SELECT dbms_json_schema.describe('EMPLOYEES', 'HR', 'JOB_ID') FROM dual;

{ 
   "title": "JOB_ID",
   "description" : "Current job of the employee; foreign key to job_id column of the jobs table. 
                    A not null column.", 
   "extendedType" : "string", 
   "maxLength" : 10
}
```
### Generating validation report
We propose a function to generate the schema validation report. The report contains information on the status of validation, i.e. if validation succeeded or failed, and the reasons for failure. Here is an example:

```
SELECT dbms_json_schema.validate_report('{"a" : 1}', '{"type": "object"}') AS report FROM dual;

REPORT
--------------------
{
  "valid" : true,
  "errors" :
  [
  ]
}
```

