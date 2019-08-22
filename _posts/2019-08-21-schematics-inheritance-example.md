---
layout: post
title:  "A Quick Tip For Inheritance"
date:   2019-08-21 22:08:03 -0700
categories: python bug
---

Let's say I have a 3rd party API that POSTs some data to my Python web service via a webhook. That means that I don't control the format of the data POSTed, but I still want to define a schema based on the 3rd party docs. This will let me know if the format changes suddenly, and lets me reason about the data in my codebase more easily. In this story, I'm using [Schematics](https://schematics.readthedocs.io) to define a schema to validate incoming data on my service's webhook endpoint.

The 3rd part service POSTs an "event". Here's an example request body loaded from JSON into a Python dictionary:

{% highlight python %}
event_data = {
    "published"  : "2011-11-03 18:21:26",
    "event_type" : "data_changed"
    "payload"    : "{\"id\": \"62d87196-9ee4-452f-bf72-3dd7f2720307\", \"origin\": \"my_service\"}"
}
{% endhighlight %}


Let's define some [Models](https://schematics.readthedocs.io/en/latest/usage/models.html) to represent the heirarchical structure of the 3rd party data:

{% highlight python %}
from schematics.models import Model
from schematics.types import UUIDType, StringType, DateTimeType, ModelType

class EventPayload(Model):
    id = UUIDType(required=True)
    origin = StringType(required=True)

class Event(Model):
    published = DateTimeType(required=True)
    event_type = StringType(required=True)
    payload = ModelType(EventPayload, required=True)
{% endhighlight %}

Using these Models, we can validate the incoming data and make sure everything wo-

{% highlight python %}
>>> Event(raw_data=event_data).validate()

Traceback (most recent call last):
  File "/Applications/PyCharm CE.app/Contents/helpers/pydev/_pydevd_bundle/pydevd_exec2.py", line 3, in Exec
    exec(exp, global_vars, local_vars)
  File "<input>", line 1, in <module>
  File "/Users/e/dev/schematics-example/venv/lib/python3.6/site-packages/schematics/models.py", line 235, in __init__
    app_data=app_data, **kwargs)
  File "/Users/e/dev/schematics-example/venv/lib/python3.6/site-packages/schematics/models.py", line 299, in _convert
    return func(self._schema, self, raw_data=raw_data, oo=True, context=context, **kwargs)
  File "/Users/e/dev/schematics-example/venv/lib/python3.6/site-packages/schematics/transforms.py", line 428, in convert
    return import_loop(cls, mutable, raw_data, import_converter, **kwargs)
  File "/Users/e/dev/schematics-example/venv/lib/python3.6/site-packages/schematics/transforms.py", line 174, in import_loop
    raise DataError(errors, data)
schematics.exceptions.DataError: {"payload": ["Input must be a mapping or 'EventPayload' instance"]}
{% endhighlight %}

... That makes sense. `event_data[payload]` is a serialized JSON string. `Model`s are constructed by going through each `field in raw_data`, and storing the result of `Type.convert(field)`. When our `Event` model tries to call `ModelType.convert(event['payload'])`, we pass in a `str` instead of a `dict`, causing the initialization to fail, resuling in a `DataError`. The quick fix here would be to pre-process the incoming data to deserialize the nested JSON fields before trying to instantiate my models.

At this point, I recognize that this would be a common problem: external APIs sometimes use serialized JSON strings as field values within resources. It would be nice to have a first-class way to define schemas on the deserialized values of these JSON strings. This seems like a great chance to add a [custom `Type`](https://schematics.readthedocs.io/en/latest/usage/types.html#custom-types) that extends the Schematics library to integrate it seamlessly.

I can do that! In my case, the most useful solution is actually a `Type` that would attempt to JSON-deserialize the provided value into a `dict` if it was a `str`, otherwise to pass the `dict` to the nested model as usual. That means that I'm basically adding a pre-processing step to `ModelType`.

After a bit of reading docs + docstrings + debugging examples, I find that the `ModelType.convert()` method is responsible for creating new nested Models, so I'd need to override it

Here is my first attempt:

{% highlight python %}
class JsonLoadingModelType(ModelType):
    def convert(self, value, context=None):
        if isinstance(value, str):
            try:
                value = json.loads(value)
            # We should catch the JSON specific errors so client code only needs to catch 1 type
            except JSONDecodeError:
                raise DataError({'payload': 'Input string could not be decoded as valid JSON'})
        super().convert(value, context=context)

# Let's redefine this; notice the changed type for payload
class Event(Model):
    published = DateTimeType(required=True)
    event_type = StringType(required=True)
    payload = JsonLoadingModelType(EventPayload, required=True)
{% endhighlight %}


Giving it a try:

{% highlight python %}
>>> Event(event_data).validate()

Traceback (most recent call last):
  File "/Applications/PyCharm CE.app/Contents/helpers/pydev/pydevd.py", line 2060, in <module>
    main()
  File "/Applications/PyCharm CE.app/Contents/helpers/pydev/pydevd.py", line 2054, in main
    globals = debugger.run(setup['file'], None, None, is_module)
  File "/Applications/PyCharm CE.app/Contents/helpers/pydev/pydevd.py", line 1405, in run
    return self._exec(is_module, entry_point_fn, module_name, file, globals, locals)
  File "/Applications/PyCharm CE.app/Contents/helpers/pydev/pydevd.py", line 1412, in _exec
    pydev_imports.execfile(file, globals, locals)  # execute the script
  File "/Applications/PyCharm CE.app/Contents/helpers/pydev/_pydev_imps/_pydev_execfile.py", line 18, in execfile
    exec(compile(contents+"\n", file, 'exec'), glob, loc)
  File "/Users/e/dev/schematics-example/src/example.py", line 46, in <module>
    Event2(event_data).validate()
  File "/Users/e/dev/schematics-example/venv/lib/python3.6/site-packages/schematics/models.py", line 258, in validate
    partial=partial, convert=convert, app_data=app_data, **kwargs)
  File "/Users/e/dev/schematics-example/venv/lib/python3.6/site-packages/schematics/models.py", line 299, in _convert
    return func(self._schema, self, raw_data=raw_data, oo=True, context=context, **kwargs)
  File "/Users/e/dev/schematics-example/venv/lib/python3.6/site-packages/schematics/validate.py", line 67, in validate
    raise DataError(errors, data)
schematics.exceptions.DataError: {"payload": ["This field is required."]}
{% endhighlight %}

huh... checking the stacktrace, it seems like the inner `EventPayload` model was created, but its fields were not filled in. At this point, I have to jump into my PyCharm debugger to step through and determine what's wrong (I highly recommend this tool! it's amazingly powerful!)

It turns out: I broke a cardinal rule of inheritance. I created a subtype that overrides one of it' parents methods: `.convert(...)`. But the parent method is [complex](https://github.com/schematics/schematics/blob/master/schematics/types/compound.py#L150-L162), and I really just wanted to massage the input a little. So I adopted a _delegation_ pattern. I prepped the input, but delegated the real work to the `super` type.

What did I fail to do? **Use the `return` value of the `super` method call!** My `JsonLoadingModelType.convert()` implementation implicitly returns `None`. That does not match the signature of the super type! by simply modifying the implementation to return the result of the call to the parent type:

{% highlight python %}
class JsonLoadingModelType(ModelType):
    def convert(self, value, context=None):
        if isinstance(value, str):
            try:
                value = json.loads(value)
            # We should catch the JSON specific errors so client code only needs to catch 1 type
            except JSONDecodeError:
                raise DataError({'payload': 'Input string could not be decoded as valid JSON'})
        # fixing our mistake...
        return super().convert(value, context=context)
{% endhighlight %}

Now it works!

{% highlight python %}
>>> Event2(event_data).to_primitive()

{'published': '2011-11-03T18:21:26.000000', 'event_type': 'data_changed', 'payload': {'id': 'b7887233-bde3-432b-a2f0-3e1cf739a819', 'origin': 'my_service'}}
{% endhighlight %}


The cool thing here is that even if the parent type's method return type was `None`, returning the result of that call would still be correct! The lesson is:

## If your subtype's method delegates to it's parent type's implementation, make sure to **always** return the result!

It's a simple lesson, but it makes sense and can be applied as a blanket rule to form good habits!

