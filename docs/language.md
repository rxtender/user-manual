The RxTender IDL syntax is heavily inspired from the
[rust](https://www.rust-lang.org) language.

!!! attention "unstable"
    The syntax is not stable yet. Everything documented here can
    change in the future.

## Defining an Item and an Error

Item and error types are declared as structures:

    struct Status {
        id :string;
        value :int32;
    }

In this example id and value are two fields of the Status structure. This can
can then be used to send items or notify errors on a stream.

## Scalar Types

| Type  | Definition                        | Default value |
|:------|:----------------------------------|:--------------|
|bool   | A boolean                         | false         |
|double | A double precision real number    | 0.0           |
|i32    | A signed 32bits integer           | 0             |
|u32    | An unsigned 32bits integer        | 0             |
|i64    | A signed 64bits integer           | 0             |
|u64    | An unsigned 32bits integer        | 0             |
|string | An UTF8 string                    | empty string  |


## Defining A Stream

A stream is declared the following way.

    stream State() -> Stream<I, E>;

where State is the name of the Stream, "I" is the type of the items, and E the
type of the error. I and E must be declared as structs. A stream can also accept
creation arguments:

    stream State(foo: u32, bar: u32) -> Status;

## Naming convention

items and streams are CamelCase.
