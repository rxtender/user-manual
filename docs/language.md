The RxTender IDL syntax is heavily inspired from the
[rust](https://www.rust-lang.org) language.

!!! attention "unstable"
    The syntax is not stable yet. Everything documented here can
    change in the future. Moreover, some features are not implemented yet.

## Defining an Item

    item Status {
        id :string;
        value :int32;
    }

## Adding Comments

        //
        /* */


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

## Lists

    item StatusList {
        list :list<Status>;
    }

## Defining A Stream

    stream State() -> Status;

A stream can have the same name than an item. A stream can accept arguments used for its creation:

    stream State(foo: u32, bar: u32) -> Status;

## Naming convention

items and streams are CamelCase.
