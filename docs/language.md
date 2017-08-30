
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

A stream can have the same name than a message

Any message that

## Naming convention

items and streams are CamelCase.
