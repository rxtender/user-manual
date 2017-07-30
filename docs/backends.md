
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
        },
        "required": ["what", "streamType"]
    }

example:

    {
        "what": "create",
        "streamType": "ClockTickStream"
        "args": []
    }

Creation success

example:

    {
        "what": "createAck",
        "streamId": 42
    }

Creation failure

example:

    {
        "what": "createNack",
        "streamId": 42,
        "reason": "nomem"
    }

## Python
