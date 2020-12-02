Sometimes you have multiple application that need communicate between each other. You can build this communication synchronous and asynchronous. In a sync communication, the caller service waits for a response before sendings the next message. In an asynchronous communication the caller service send messages without waiting for a response. 

A few weeks ago I need to build a solution to send alert messages to clients of the company, something like a broadcast service. I only had a core service that had access to the database, but this service had many responsabilities alredy and add a new feature to send a big amount of messages could overload it. So I decided to add a small feature, the core service gets from database all the users that should receive the message and build the message content, and sends theses to another service, that actualy sends the messages.

This second service that only sends message could be used for multiple others application, like an alert service to notify the clients of promotions, or a remider service of schedule appointments. And all the service could only filter the users that should receive de message, build the content of the message, and sends to the broadcast service.

In my problem I had multiple microservices publishesing or sending messages to a channel, and one service that is listens for messages on one channel. So looking for solutions, I found many technologies like Redis, RabbitMQ and KafKa. For this tutorial let's build a simple example using Redis Pub/Sub.


### The subscriber

The first application that we're going to build is the subscriber. Our subscriber will listen a channel, when a message arived it will get notified. For this tutorial, we'll send a sms messager using the twilio library.


```python
import os
import redis
import json

from twilio.rest import Client
from multiprocessing import Process 

redis_conn = redis.Redis(charset="utf-8", decode_responses=True)

def sub(name: str):
    pubsub = redis_conn.pubsub()
    pubsub.subscribe("channel")

    for message in pubsub.listen():
        message = json.loads(message)

        if message.get("type") == "message":
            data = message.get("data")
            print("%s : %s" % (name, data))

            account_sid = os.environ.get("TWILIO_ACCOUNT_SID")
            auth_token = os.environ.get("TWILIO_AUTH_TOKEN")

            body = data.get("message")
            from_ = data.get("from")
            to = data.get("to")

            client = Client(account_sid, auth_token)
            message = client.messages.create(from_=from_, to=to, body=body)
            print("message id: %s" % message.sid)


if __name__ == "__main__":
    Process(target=sub, args=("reader1",)).start()
```

Let's undestand our script. After we imported a few libraries, we created a redis connection, passing `decode_responses` as True, so the client will decode the response using the `encoding` option.

```python
redis_conn = redis.Redis(charset="utf-8", decode_responses=True)
```

Now we need to instantiate a pubsub object and subscribe to a channel. 

```python
def sub(name: str):
    pubsub = redis_conn.pubsub()
    pubsub.subscribe("channel")

    ...
```

We also can subscribe using a pattern. For example, if we had mutiple channels like `channel-broadcast`, `channel-alert`, `channel-remider`. We can subscribe all channels that starts with `channel-` using the pattern `channel-*`, i.e `pubsub.subscribe("channel-*")`. 

Next we need to continuosly listening to subscribed channel. We can do this using the method callend `listen()`, this is a generator that blocks the execution and waits for a new message on the channel.

```python
def sub(name: str):
    pubsub = redis_conn.pubsub()
    pubsub.subscribe("channel")

    for message in pubsub.listen():
        ...
```

We can only publish message of type string, bytes or float on the channel, but when the subscribe gets a message from the channel, it cames as a dictionary object. For example, if we publish `hello`, the subscribe gets:
```json
{
    "type": "message", 
    "pattern": None, 
    "channel": b"broadcast", 
    "data": b"hello"
}
```

There are four keys:
- `type`: the type of message. There are sex type: `subscribe`, `unsubscribe`, `psubscribe`, `punsubscribe`, `message`, `pmessage`.
- `pattern`: in our example, the pattern is `None`, it's the default value. But if we use the pattern subscribe, this field will store the pattern used, ie. `channel-*`.
- `channel`: the channel name
- `data`: the actual messa published on the channel. 

In this tutorial we expected the content of `data` with a json structure. So we need serialize the object before publish and deserialize it on the subscriber. We can use `json.loads` to takes the string message and returns a json object. An example of message published is:
```json
{
    "message": "hello", 
    "from": "+12028165004", 
    "to": "+5584981341213"
}
```

The `message` field is the content of the message, the `from` field informs our phone number, and the `to` field define the phone number that we should send a message.

```python
def sub(name: str):
    pubsub = redis_conn.pubsub()
    pubsub.subscribe("channel")

    for message in pubsub.listen():
        if message.get("type") == "message":
            data = json.loads(message.get("data"))
            print("%s : %s" % (name, data))

            body = data.get("message")
            from_ = data.get("from")
            to = data.get("to")
```

Now that we get all the data that we need to send a message, let's read the environment variable `TWILIO_ACCOUNT_SID` and `TWILIO_AUTH_TOKEN`. Next we create a Twilio `Client` and send the sms using the `messages.create` function.


```python
import os
import redis
import json

from twilio.rest import Client
from multiprocessing import Process 

redis_conn = redis.Redis(charset="utf-8", decode_responses=True)

def sub(name: str):
    pubsub = redis_conn.pubsub()
    pubsub.subscribe("channel")

    for message in pubsub.listen():
        message = json.loads(message)

        if message.get("type") == "message":
            data = message.get("data")
            print("%s : %s" % (name, data))

            body = data.get("message")
            from_ = data.get("from")
            to = data.get("to")

            account_sid = os.environ.get("TWILIO_ACCOUNT_SID")
            auth_token = os.environ.get("TWILIO_AUTH_TOKEN")

            client = Client(account_sid, auth_token)
            message = client.messages.create(from_=from_, to=to, body=body)
            print("message id: %s" % message.sid)


if __name__ == "__main__":
    Process(target=sub, args=("reader1",)).start()
```

Now we can make a simple test of our code. Let's run the `sub` function using `Process` from `multiprocessing`. We need to use the `Process` here because the event loop generated when we call `listen()` is blocking, it's meaing we can't to do anything else instead of waiting for new messages. For this simple example the blocking property is not a problem, but maybe in a real scenario it could be.


### The publisher


```
import redis
import json
 
redis_conn = redis.Redis(charset="utf-8", decode_responses=True)

def pub():
    data = {"message": "hello", "from": "+12028165004", "to": "+5584981341213"}
    redis_conn.publish("channel", json.dumps(data))
    return {}
```

