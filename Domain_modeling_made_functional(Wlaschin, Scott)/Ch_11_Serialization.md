
# Chapter 11: Serialization 

Our bounded context receives and sends things to the outside that has no knowledge of our internal domain model types.
This we must convert between these and a communication format like JSON, XML or protocol buffers. 
We also need to keep track of the internal state of a workflow using a database. 

## Persistence vs. Serialization
Persistence - state that outlives the process that created it.
Serialization - the process of converting from a domain-specific representation to a representation that can be persisted easily

Persistence could mean stored as a file or in a queue; it doesnt have to be a database.
The lifetime of the persisted data can vary — it could be kept around for just a few seconds (such as in a queue) or it could be kept around for decades (such as in a data warehouse).

## Designing for serialization

The trick to pain-free serialization is to convert your domain objects to a type specifically designed for serialization—a Data Transfer Object—and then serialize that DTO instead of the domain type.
For deserialization, we do the same thing in the other direction.
Deserialization into a DTO should always succeed unless the underlying data is corrupt - Any kind of domain-specific validation (such as validating integer bounds for an OrderQty or checking that a ProductCode
is valid) should be done in the DTO-to-domain-type conversion process, inside the bounded context, where we can control error handling.

## Connecting the serialization code to the workflow 
The *deserialization* step is added at the start of the workflow and the *serialization* step at the end of the workflow.

The function signatures for the deserialization step might look like
```

type JsonString = string
type MyInputDto = ...
type DeserializeInputDto = JsonString -> MyInputDto
type InputDtoToDomain = MyInputDto -> MyInputType
```
The serialization step might look like :
```
type MyOutputDto = ...
type OutputDtoFromDomain = MyOutputType -> MyOutputDto
type SerializeOutputDto = MyOutputDto -> JsonString
```

I.e. the function exposed to the outside world is a deserialization/serialization sandwich of the workflow.
```
let workflowWithSerialization jsonString =
	jsonString
	|> deserializeInputDto
	|> inputDtoToDomain 
	|> workflow 
	|> outputDtoFromDomain
	|> serializeOutputDto 
```
### DTOs as a contract between bounded contexts 

The contract between our bounded context and the outside world is the incoming commands are triggered by the outputs of other bounded contexts, and the events that our workflow emit that becomes the inputs for other
bounded contexts

The serialized format of these events and commands, which are DTOs, should only be changed carefully or not at all. You should always have complete control of the serialization format and avoid library magic. 

## A complete serialization example 
See p. 224
Create module Dto with types where all the fields are primitives.
```
module Dto =
	type Person = {
		First: string
		Last: string
		Birthdate : DateTime
}

	module Person = 
		let fromDomain (person:Domain.Person) :Dto.Person = ...
		let toDomain (dto:Dto.Person) :Result<Domain.Person,string> = ...
```

from/To mapping is straightforward as we just go from simple types to domain types and vice versa. 

### Wrapping the JSON serializer 
Generally we use a library for serializing. However, we might want to wrap it to make it more functional-friendly, i.e. return a Result type instead of simply throwing an exception if something goes wrong. 

```
module Json =
	open Newtonsoft.Json
	let serialize obj =
		JsonConvert.SerializeObject obj
	let deserialize<'a> str =
		try
			JsonConvert.DeserializeObject<'a> str
			|> Result.Ok
		with
		| ex -> Result.Error ex
```

### A complete serialization pipeline 
Composing the serialization pipeline is straightforward as they dont involve any Result types, but the Json.deserialize and the PersonDto.fromDomain can return Result values during deserialization

Use Result.mapError to convert the potential failures to a common choice type and then use a result expression to hide the errors (p. 201) - see p. 227

Alternatively, allow the deserialization code to throw exceptions. The best approach depends on whether you want to handle deserialization errors as an expected situation or as a “panic” that crashes the entire pipeline.
Consider: 
How public is your API?
How much you trust the callers, 
How much information you want to provide the callers about these kinds of errors?

### Working with Other Serializers
If not using Newtonsoft for serializer, it might not work out the box. To serialize a record type using the DataContractSerializer (XML) or DataContractJson-
Serializer (JSON), you must decorate your DTO type with DataContractAttribute and DataMemberAttribute. 

One of the benefits of keeping the domain type separate from the Dto type is you don't have to annotate your domain type here. 

The CLIMutableAttribute attribute emits a (hidden) parameterless constructor, often needed by serializers that use reflection.

If you're only working with other F# components, you can use a F#-specific serializer such as FsPickler or Chiron. However, this creates coupling between the bounded contexts in that they all must use the same programming language.

### Working with Multiple Versions of a Serialized Type
The DTO is the contract with other bounded contexts. Sometimes we need to change the internal domain model by adding, removing or renaming fields which may affect the DTO and break the contract. 
To prevent this, you might need to support multiple DTO versions of a DTO type at the same time.
See [Greg Young - Versioning in an Event Sourced System](https://leanpub.com/esversioning) for discussion of different approaches

## How to translate domain types to DTOs

**Single-Case Unions**
Use underlying type 

**Options**
Reference types: Replace None with null. 
Value type t: Nullable<t>, e.g. Nullable<int> 

**Records**
Records are translated to records where domain types are replaced by simple types 
 
**Collections**
Lists, sequences, and sets: arrays
Maps and other complex collections: Depends on serialization format 
- If JSON, map translates directly to a JSON object 
- Else you may need to create a special representation. 
-- A map might be represented in a DTO as an array of records, where each record is a key-value pair.
-- Alternatively, a map can be represented as two parallel arrays that can be
zipped together on deserialization.
 
**Discriminated Unions Used as Enumerations**
 Unions where every case is just a name with no extra data can be represented by .NET enums, which are generally represented by integers when serialized.
 when deserializing, you must handle the case where the .NET enum value is not one of the enumerated ones (p.232)
 Alternatively, you can serialize an enum-style union as a string, using the name of the case as the value, but renaming can lead to issues.
 
**Tuples**
Will probably need to be represented by a specially defined record; tuples are not supported in most serialization formats.
 
**Choice Types**
Can be represented as a record with a “tag” that represents which choice is used and then a field for each possible case that contains the data associated with that case. When a specific case is converted in the DTO, the
field for that case will have data and all the other fields, for the other cases, will be null/empty list (p. 233) 

When serializing you just need to convert the appropriate data for the selected case and set the data for all the other cases to null.
When deserializing, we match on the “tag” field and then handle each case separately. We must always check that the data associated with the tag is not null before we attempt deserialization

**Serializing Records and Choice Types Using Maps**
An alternative serialization approach for compound types (records and dis-
criminated unions) is to serialize everything as a key-value map; this aligns well with the JSON object model.
The advantage of this approach is that there’s no “contract” implicit in the
DTO structure;a key-value map can contain anything so it promotes highly decoupled interactions

The downside is that there’s no
contract at all! That means that it’s hard to know when there is a mismatch
in expectations between producer and consumer.

Serialization: Create a list of key/value pairs and use the built-in function dict to build an IDictionary from them. 

The IDictionary uses obj as the type of the value, so all the values in the record must be explicitly cast to obj using the upcast operator :> .

Deserialization: For each field we need to look in the dictionary to see if it’s there; if present, retrieve it and attempt to cast it into the correct type.

**Generics**
If the serialization library supports generics, then you can create DTOs using generics as well.
Else you’ll have to create a special type for each concrete case.