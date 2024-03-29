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

* `GET` is available only to simple contracts; `POST` does not have such restriction
* no basic auth
* in `POST` requests data is always sent in body as JSON or MsgPack
* SSL must be terminated on reverse-proxy level (nginx etc.)

### tl;dr

LGNC is not a framework, but a convention on sending requests over networks (not just HTTP): it defines
request/response envelope format, ensures that all participants know signatures of all contracts
(i.e. no blackboxes/gentlemen's agreements), and guarantees that assertions above will be rock hard
regardless of concrete language implementation.

### Use cases

There are two main use cases for LGNC:

1. iOS app + backend: say you have a mobile application<sup id="a1">[*](#f1)</sup>, and most probably you would want
it to communicate with your backend. Usually it's a blackbox backend on PHP/Python/Java/.NET/Go, and you send
requests to it using Alamofire/URLSession/Swift-NIO. The problem here is that your contract with backend may break
at any time, not to mention that you have to map every request/response to a struct/class, and sometimes response
might be in a slightly different format (remember that "do we send booleans as true/false or 0/1?" discussion).
LGNC allows you to just dodge all that problems, proceeding directly to business tasks. All you have to negotiate
with backend team is a contract signature in clean and simple YAML format, after that you generate boilerplate
contracts for your app and execute them directly (as code, not JSON or whatever), without having to bother about
transport or format details, and in the meantime BE team does precisely the same thing: they generate boilerplate
contracts for BE, guarantee them and just hook up business logic to them. You will never catch an "oof, I expected
this field to be integer, and it actually is, but for some reason it is in quotes, and because of that Codable failed
to decode response entity, _le sigh_". Even more, if you care about timings (we all hate slow webservices/apps)
you don't have to communicate with your backend via HTTP, because LGNC offers you a much simpler and compact TCP
protocol LGNP, which does absolutely everything you would _generally_ want from HTTP, just in a more compact way.
Still, if certain contract must be available from HTTP, it is totally not a problem, see details below on that.

2. Second most common use case for LGNC is a microservice system. Contrary to popular belief, microservice
architecture is not an enterprise thing (a webstore may consist of auth service, merchant service, billing service,
customer support service, and God knows what else), but people are often scared by apparent complexity of it
(generally all API endpoints are poorly documented somewhere deep in Confluence, and basically are just
gentlemen's agreements), whereas microservice arch is a quite reasonable approach for many (but not all) cases.
And LGNC might be _the_ tool for building a reliable, maintainable and robust multiservice stack:
you define all your services and contracts in one place that is common for all services (again, your services
aren't necessarily written in one language, it just a hub, source of truth), generate skeleton contracts for the
service you will implement, as well as contracts for services you want to communicate with, and then just execute
those contracts as if it's a module in your service, and not a dedicated service written in a different language on
a remote machine.

<sub><b id="f1">*</b> It's not necessarily just an _iOS_ app. As we add more languages (like Kotlin),
Android apps would also be able to use LGNC absolutely the same way as iOS/macOS ecosystem
(and Linux) can do now. [↩](#a1)</sub>

## Supported languages

Currently LGNC is fully supported in Swift 5.2 by [LGNC-Swift](https://github.com/1711-Games/LGNC-Swift).  Closest
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
only one implementation — [LGNC-Swift](https://github.com/1711-Games/LGNC-Swift).

### Complete schema example

Let's consider this most comprehensive example containing two services with one and two contracts respectively, and
a shared section with two entities:

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
                Cumulative:
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
                IdenticalWith: password1
            boolField: Bool
            customField: Baz
            listField: List[String]
            listCustomField: List[Baz]
            mapField: Map[String:Bool]
            mapCustomField: Map[String:Baz]
        Response:
          Fields:
            ok: List[String]

  Second:
    Contracts:
      Simple:
        IsGETSafe: True
        Request:
          query: String
          session: Cookie
        Response:
          List[ExtendedBaz]
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
Services/Second/Contracts/Simple.yml
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
* `IsGETSafe` — indicates that contract may safely be executed via `HTTP` `GET` request (instead of `POST`) with request
  fields in URL query string. Contract request must only have `String`/`Int`/`Bool` (in `true`/`false` string form)
  fields (including `Cookie` fields in HTTP headers), otherwise it cannot be marked as `IsGETSafe` (builder would fail
  during the schema validation). Default: `false`.

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

For your convinience, LGNC has some builtin entities, namely:
* `Empty` which is just an entity without any fields. Comes in handy when your contract should have empty request or
  response.
* `Cookie` which represents an HTTP cookie. If your contract is executed via HTTP, and it has a `Cookie` field in
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

A field may have an arbitrary number of validators.  All validators, with one exception, are executed sequentially
(hence, order matters), failing at first error, therefore it is recommended to put lightweight validators on top,
leaving heavy validators (`Regex`, `Callback`) to the bottom of the list.

All validators (except `Cumulative`) may have a `Message` entry with an error message if it fails.

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
  tuple (pair) of error code (`Int`) and error message (`String`), OR a list of such tuples (multi-error). `Errors` is
  a list of objects with following entries: `Code` of type `Int` and `Message` of type `String`.  Additionally,
  `Error` may contain `ShortName` key which holds a short name of this error.  It's useful for languages with `enum`
  data type, like Swift.  Otherwise enum key will be just an error message without whitespaces (which isn't really
  beautiful, let's admit that).  **Important**: target language implementation _must_ ensure that a callback with
  an allowed list of errors can only return listed errors.
* `Cumulative` is another special kind validator which groups all nested validators to be executed in parallel, without
  fast fail. Thus, field can return arbitrary number of errors. Comes in handy for username (or password) validation.

It is worth noting that before any fields validators are executed, LGNC will ensure that request body contains
ONLY entries stated in request schema, otherwise it will return an error
`{code: 422, message: "Input contains unexpected items: 'foo', 'bar'"}`.
After that it will build a chain of validators for each field stated in request schema. Each chain (if field is not
optional) starts with a validator which ensures that a field is present in the request body and it's of correct type,
otherwise field validation would fail with an `{code: 512, message: "Value missing"}` error (message may vary depending
on `MissingMessage` attribute of field). Thus, LGNC doesn't make a difference between absence of field in request data
or invalid field type.

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

### Don't mark destructive/non-readonly contracts as `IsGETSafe`

It's more of a general advice rather than LGNC-related, but still I, quite an experienced web-dev, caught myself
a couple of times marking non-readonly contracts as GET-safe, whereas they were write-contracts, hence they were
changing something in app state (DB or whatever), which should only be done via POST method rather than GET.

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
