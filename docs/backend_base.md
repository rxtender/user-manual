
## Serialization

### JSON

Creation message:

    {
        "title": "Create Message",
        "type": "object",
        "properties": {
            "what": {
                "description": "the message type. Value is 'create'",
                "type": "string"
            },
            "streamType": {
                "description": "The type of stream to create",
                "type": "string"
            },
            "streamId": {
              "description": "Id of the stream to create",
              "type": "number"
            },
            "args": {
              "description": "arguments used to create the stream",
              "type": "array"
            }
        },
        "required": ["what", "streamType", "streamId", "args"]
    }

example:

    {
        "what": "create",
        "streamType": "ClockTickStream",
        "id": 42,
        "args": []
    }

Creation success message:

example:

    {
        "what": "createAck",
        "streamId": 42
    }

Creation failure message:

example:

    {
        "what": "createNack",
        "streamId": 42,
        "reason": "nomem"
    }

Deletion message:

    {
        "title": "Delete Message",
        "type": "object",
        "properties": {
            "what": {
                "description": "the message type. Value is 'delete'",
                "type": "string"
            },
            "streamId": {
              "description": "Id of the stream to delete",
              "type": "number"
            }
        },
        "required": ["what", "streamId"]
    }

example:

    {
        "what": "delete",
        "id": 42
    }

Deletion success message:

example:

    {
        "what": "deleteAck",
        "streamId": 42
    }


Next item message:

    {
        "title": "Next Message",
        "type": "object",
        "properties": {
            "what": {
                "description": "the message type. Value is 'next'",
                "type": "string"
            },
            "item": {
                "description": "the serialized item",
                "type": "object"
            },
        },
        "required": ["what", "item"]
    }

example:

    {
        "what": "next",
        "item": {
            "foo": 42,
            "bar": "buzz"
          }
    }

Completed message:

    {
        "title": "Completed Message",
        "type": "object",
        "properties": {
            "what": {
                "description": "the message type. Value is 'completed'",
                "type": "string"
            },
        },
        "required": ["what"]
    }

example:

    {
        "what": "completed"
    }

Error message:

    {
        "title": "Error Message",
        "type": "object",
        "properties": {
            "what": {
                "description": "the message type. Value is 'error'",
                "type": "string"
            },
            "message": {
                "description": "A message containing information about the error",
                "type": "string"
            },
        },
        "required": ["what", "message"]
    }

example:

    {
        "what": "completed"
    }

## Python

## ES2015
