# LGNC — LGN Contracts

![LGNC Logo](./logo.png)

## About

LGNC is a simple yet powerful tool for building services.  The general idea is you define your services and contracts in
YAML format (see details below), then you generate a codebase for target platform and it does all the boring stuff for
you like routing and request validation.

Originally LGNC was developed for internal services communication (game backend, to be more specific), and the priority
was to make it real fast and compact.  Therefore a dedicated protocol has been developed — LGNP[rotocol] with a
respective server — LGNS[erver].  You may call LGNP an abridged version of HTTP, if you wish.  LGNS (hence LGNP) is a
first class citizen in LGNC ecosystem.

HOWEVER.  Let me dispel your concerns with this: LGNC does support HTTP transport, but with some limitations:

* it's always `POST` method
* [almost] no headers
* no basic auth
* no URL params (data is always sent in body as JSON or MsgPack)
* SSL must be terminated on reverse-proxy level (nginx etc.)

## Supported languages

Currently LGNC is fully supported in Swift 5.2 by [LGNKit-Swift](https://github.com/1711-Games/LGNKit-Swift).  Closest
plans are JavaScript, PHP, Java, Kotlin and Go.

## Usage

### General idea

The first thing you have to do (regardless of target language) is to define your schema.  At its simplest, a schema is a
list of your services (with contracts) and a shared section which holds entities that are common for two or more
services.  `Shared` section is optional, whereas `Services` section is mandatory (it must contain at least one service).
Every service has a `Contracts` section which contains all service's contracts.  Every contract must have `Request` and
`Response` entries, which are entities.  Every entity must have a `Fields` entry with a list of fields.

Then you do codegen for target language using [LGNBuilder](https://github.com/1711-Games/LGNBuilder) (see respective
documentation).  It looks something like this:

```bash
LGNBuilder/Scripts/generate --lang=Swift --input /your/schema --output /your/codebase
```

You can specify a list of services to generate as a final argument of command, otherwise all services will be generated.

Other useful arguments are `--dry-run` which will validate the schema and make sure that everything will work and
`--emit-schema` which will output preprocessed schema.

After you've done codegen, you're almost there: all you have to do is to _guarantee_ all contracts in the service you're
implementing.  Guarantee is a body of a contract: it's a piece of code (an anonymous function, for example) which takes
contract request (defined in schema) and request context (request details like remote address and stuff like that), and
returns response (also defined in schema, respectively).  Optionally, a guarantee body may return a meta result (just a
`Dictionary<String, String>`) along with the response.  Additionally, if contract has callback validators, you must also
guarantee those.  When all contracts are guaranteed, you may start the server (or servers, if your service serves more
than one transport), et voilà!  You're up and running.

Please refer to target language implementation for complete integration documentation and tutorials.  Currently there is
only one implementation — [LGNKit-Swift](https://github.com/1711-Games/LGNKit-Swift).

### Complete schema example

Let's consider this most comprehensive example containing two services with one contract each, and a shared section with
two entities, with one of them (`Baz`) being used by both contracts:

```yaml
Shared:
  Entities:
    Baz:
      Fields:
        One: Bool
        Two: Bool
        Three: Bool
    ExtendedBaz:
      ParentEntity: Baz
      ExcludeFields: [ One ]
      Fields:
        Two: String
        Four: Int

  DateFormat: "yyyy-MM-dd kk:mm:ss.SSSSxxx" # Optional

Services:
  First:
    # Optional KV readonly storage
    Info:
      Key1: Value1
      Key2: Value2

     # Optional, defaults to "LGNS: 1711"
    Transports:
      LGNS: 1711
      HTTP: 1337

    Contracts:
      DoThings:
        Transports: [ HTTP, LGNS ]
        ContentTypes: [ JSON, MsgPack ]
        Request:
          Fields:
            stringField: String
            intField: Int
            stringFieldWithValidations:
              Type: String
              Validators:
                Regex:
                  Expression: ^[a-zA-Zа-яА-Я0-9_\\- ]+$
                  Message: "Your input is invalid"
                MinLength: 3
                MaxLength: 24
                Callback:
                  Errors:
                    - { Code: 322, Message: First callback error }
                    - { Code: 1337, Message: Second callback error }
                    - { Code: 1711, Message: "This is seventeen eleven, yo", ShortName: E1711 }
            enumField:
              Type: String
              AllowedValues:
                - first allowed value
                - second allowed value
            optionalEnumField:
              Type: String
              DefaultNull: True
              Validators:
                In:
                  AllowedValues:
                    - lul
                    - kek
            uuidField:
              Type: String
              Validators:
                - UUID
            dateField:
              Type: String
              Validators: [ Date, Callback ]
            password1: 
              Type: String
              MissingMessage: "You must enter password"
              Validators:
                NotEmpty:
                Message: "Empty password"
            password2:
              Type: String
              Validators:
                IdenticalWith:
                  Field: password1
            boolField: Bool
            customField: Baz
            listField: List[String]
            listCustomField: List[Baz]
            mapField: List[String:Bool]
            mapCustomField: List[String:Baz]
            session: Cookie
        Response:
          Fields:
            ok: List[String]

  Second:
    Contracts:
      DoOtherThings:
        Request: Baz
        Response: Baz
```

Surely, such single-file form is comprehensive, and everything is at hand, but it's not very comfortable for reading
(managing two-space indentation is really bad for your eyes).  That's why your schema can be safely broken down into
arbitary number of pieces and organised as a file structure like this:

```
Shared/__root__.yml
Shared/Entities/Baz.yml
Services/First/__root__.yml
Services/First/Contracts/DoThings.yml
Services/Second/Contracts/DoOtherThings.yml
```

Each file here represents a nested level in YAML schema.  Every path component means exactly what you think it means :)
You don't have to preserve indentation in each file, it will be added automatically during preprocessing.
`__root__.yml` is a magic filename which means that its contents should be applied at current level root.  In this
particular case we will put there the date format:

```
DateFormat: "yyyy-MM-dd kk:mm:ss.SSSSxxx"
```

NB: `DateFormat` is used for dates validation.  If it's not specified in `Shared`, it defaults to
`yyyy-MM-dd kk:mm:ss.SSSSxxx`.

`Services/First/__root__.yml` contains supporting contract info:

```yaml
Info:
  Key1: Value1
  Key2: Value2

Transports:
  LGNS: 1711
  HTTP: 1337
```

Simple and convenient, isn't it?

NB: if you have your schema in this form (why wouldn't you do that though?), you can always print out complete assembled
schema using LGNBuilder flag `--emit-schema`:

```bash
LGNBuilder/Scripts/generate --emit-schema --lang=Swift --input /schema --output /codebase
```

### Service format

Service must contain an entry `Contracts` with a map (dict) of all contracts.

Service may contain following additional entries:

* `Info` is a simple key-value storage for using in contracts bodies.  Can be convenient sometimes.
* `Transports` is a map of allowed service transports and respective ports they listen on.  Service may be configured
  for different ports at runtime.  Default: `LGNS: 1711`.

### Contract format

Contract must contain entries `Request` and `Response` which are request and response entities respectively.

Contract may contain following additional entries:

* `URI` — universal resource identifier under which this contract will be reachable.  Default: contract name.
* `Transports` — a list of allowed transports for contracts.  Some contracts have to be internal, and for that purpose
  limiting contract to be executed only via LGNS transport (which allows high security level) is quite handy.  Default:
  all service transports.
* `ContentTypes` — a list of allowed content types for contract.  Default: `JSON`.

### Entity format

Entity must contain entry `Fields` with all fields.

Entity may contain following additional entries:

* `ParentEntity: SharedEntityName` which means that this entity will copy all fields from `SharedEntityName` entity
  (from `Shared.Entities` section), and add own `Fields` (overwriting existing fields with same names) after copying
  fields from parent entity.
* `ExcludeFields: [ fieldName1, fieldName2 ]` is relevant only when  `ParentEntity` is specified, it means that when
  copying fields from parent entity, listed fields will be ignored.  To summarize: first we take fields from
  `ParentEntity` without those listed in `ExcludeFields`, and then add fields listed in `Fields`.  Thus `ExtendedBaz`
  entity will look like this after preprocessing:

```yaml
ExtendedBaz:
  Fields:
    Two: String
    Three: Bool
    Four: Int
```

#### Builtin entities

For your convinience, LGNC has some builtin entities, such as:
* `Empty` which is just an entity without any fields. Comes in handy when your contract should have request or response.
* `Cookie` which represents a HTTP cookie. If your contract is executed via HTTP, and it has a `Cookie` field in
  request, it will lookup it in headers first (which are transparently passed to contract as meta, along with request
  entity), and if it's missing in headers, it will lookup it in request, as usual. If your contract is executed via
  LGNS, it will lookup it directly in meta, and then in request. If your contract response has `Cookie` field, and it's
  executed via HTTP, the cookie will transparently be set to response headers (and meta). Still, it will be available in
  response body, as contracts are always strict on request/response format.

See language implementation documentation for more details about both these entities.

### Field format

As you might've noticed in examples above, field can be in two forms: string and object.  String form means that it's
just a bare type with all other properties in default (empty) state.

#### Allowed types

* `String` — a UTF-8 string
* `Int` — a 64-bit LE integer
* `Float` — a 32-bit LE float
* `Bool` — simple 1-bit `true`/`false` boolean
* `Cookie` — an HTTP cookie.
* `List[ValueType]`— an ordered list of values of given type
* `Map[KeyType:ValueType]` — an unordered map (dictionary).  `KeyType` must be `String` or `Int`.
* Custom type from `Shared.Entities` section

#### Object entries

Object field must contain a `Type` entry.  Additionally, a field can have following entries:

* `MissingMessage` is an error message if field is missing in request (relevant for non-optional fields only).  Keep in
  mind though that _missing_ isn't equal to _empty_, see `NotEmpty` validator.  Default: `Value missing`.
* `IsMutable: true` (available in languages which have mutable/immutable variables) makes this field mutable.  Default:
  `false`.
* `IsNullable: true` makes this field optional, but if it's specified in request, all validators are executed.  Default:
  `false`.
* `AllowedValues: [ foo, bar, baz ]` (a shortcut for `AllowedValues` validator, see below) specifies a list of allowed
  values for field (applicable only for `String` fields).
* `NotEmpty: true` (a shortcut for `NotEmpty` validator) if this field is present in request, but is empty, an error
  will be returned (applicable only for `String` fields).
* `DefaultEmpty: true` (experimental feature) if field is missing in request, assumes its default value (zero for `Int`,
  empty string for `String` etc) or `nil` (`null`) if field is optional.  Default: `false`.
* `DefaultNull: true` a shortcut for setting `IsNullable` and `DefaultEmpty` to `true`.  Default: `false`.
* `Validators` a map of validators (see below).

#### Validators

A field may have an arbitrary number of validators.  All validators are executed sequentially (hence, order matters),
failing at first error, therefore it is recommended to put lightweight validators on top, leaving heavy validators
(`Regex`, `Callback`) to the bottom of the list.

Every validator may have a `Message` entry with an error message if it fails.

Field may have following validators:

* `NotEmpty` is a validator without params, it just ensures that input string is not empty.  It is recommended to use it
  in field body, rather than in validators list.
* `UUID` is a validator without params, it ensures that input string is an UUID in hex form
  (`8011b1fb-74b5-4d23-b476-1f3c0e2edae8`, case insensitive)
* `Regex` does validation against a regular expression in `Expression` entry
* `In` ensures that field value is one of `AllowedValues` values.  It is recommended to use it as `AllowedValues` in
  field body, rather than in validators list.
* `MinLength` / `MaxLength` validates input string against minimum or maximum characters length specified in `Length`
  param (not ASCII symbols, but rather UTF-8 characters, hence string `💖É💖` must be treated as 3 symbols, not 10).
  Additionally, this validator may be in int form rather than in object: `MinLength: 3`.
* `IdenticalWith` ensures that target `Field` value is identical with this one.  Main use case is passwords, of course.
  This validator may be in string form: `IdenticalWith: password1`.
* `Date` is a date validator. It ensures that input string date is in `Format`/`Shared.DateFormat` format. If format
  is not specified in `Shared.DateFormat`, it defaults to `yyyy-MM-dd kk:mm:ss.SSSSxxx`.
* `Callback` is a special kind of validator.  It allows you to specify custom validation for the field.  Use cases are
  username/email lookup in database, custom complex validations, calling external services for confirmation etc.  There
  are two types of a callback validator: the one with arbitrary error messages, and the one with a hardcoded list of
  allowed errors.  If you don't specify `Errors` key, it's treated as simple callback validator.  Its body must return a
  tuple (pair) of error code (`Int`) and error message (`String`).  `Errors` is a list of objects with following
  entries: `Code` of type `Int` and `Message` of type `String`.  Additionally, `Error` may contain `ShortName` key which
  holds a short name of this error.  It's useful for languages with `enum` data type, like Swift.  Otherwise enum key
  will be just an error message without whitespaces (which isn't really beautiful, let's admit that).  **Important**:
  target language implementation _must_ ensure that a callback with an allowed list of errors can only return listed
  errors.

## Anti-patterns

### Don't use shared entities outside of contract

**tl;dr: LGNC isn't intended to describe your business logic models, it's _a layer_ between outer world and your
business logic**

It may seem to be a good idea, for example, to define an ultimate shared entity named `User` and then use it throughout
your app as a user model, but in reality it will eventually be re-assembled as someone (maybe even you) edits the
schema, and happy you if your code would just stop compiling, because of broken definitions.  Worst case scenario is
when you don't even notice anything, but your business logic is already silently corrupted, and something isn't working
as expected.

Also let's not forget that schema format is rather limited and it can't model _everything_ you would like to express
with it (for example, even though it has something like an enum (`AllowedValues` attribute), it's not translated into
actual `enum` in Swift because of _reasons_, however you would still like it to be an `enum`).

## FAQ

### Why not OpenAPI/Swagger? How is it different?

OpenAPI is a total overkill for 99% of web applications, including quite complicated ones.  From my personal experience,
I've never yet worked on a project that wouldn't be hundred percent satisfied with current (pre-release) LGNC
featureset.

Don't get me wrong, OpenAPI is a great and mature tool, but it's just too much, and you can't really use the bare
minimum of it while preserving the acceptable level of readability of schemas (manifestos, you name it).  LGNC, on the
other hand, does precisely that: it's ultra laconic and offers THE featureset you will ever need both in your pet
projects and real-world production systems.

Of course, you may find it lacking certain features here and there, but this is what I meant earlier: you just entered
that 1% that wouldn't be satisfied by LGNC.  And TBH it's totally not what we aim for: we can't and we won't [ever be
able to] completely satisfy everyone.

### Still don't get it.  Is it better than gRPC then?

Here is a flashback.  I started building LGNC (and LGNKit in general) back in 2015, when Swift went opensource.  For
obvious reasons, there wasn't anything at all yet at that time.  When we got gRPC/OpenAPI for Swift, it was way too late
for me :)

I'm not saying that it's better than gRPC or OpenAPI, I'm trying to say that it does everything I need it to do, no
more, no less.  This is what makes it better _personally_ for me.
