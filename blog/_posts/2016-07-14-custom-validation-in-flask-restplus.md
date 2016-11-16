---
layout: post
author: Matt Dickinson
title:  "Custom Validation in Flask RESTPlus"
date:   2016-07-14 17:45:00
tags:
  - code
  - python
  - restplus
---

I was pulling my hair out yesterday afternoon and this morning trying to figure out how to get [Flask RESTPlus](http://flask-restplus.readthedocs.io/en/stable/) to return a different error code
for validation errors. Out of the box, it will return a 400 code with the validation error messages (as it should). Unfortunately, I was tasked 
with writing an endpoint that would fulfill a previously determined contract; this contract expects either a 200 or a 500.

## Custom validation

One thought I had was to disable the RESTPlus validation and do my own validation. You can use the `validate=False` argument in `Namespace.expect()` to supply an expected request format 
(for Swagger documentation) while ignoring validation of that model.

```python
    @ns.expect(myRequestModel, validate=False)
    def post(self):
        #Do custom validation and return a 500 if it fails
```

Unfortunately, this is a lot of wheel reinventing for very little gain.

## Overriding RESTPlus validation

After pouring over the documentation and [source code](https://github.com/noirbizarre/flask-restplus), I eventually arrived at the following solution. Of course, there may be better ways to do it,
but this was the way I finally got it to work.

```python
class MyResource(Resource):

    # Override validate_payload method from Resource class
    def validate_payload(self, func):
        try:
            super(Resource, self).validate_payload(func)
        except BadRequest as e:
            abort(status.HTTP_500_INTERNAL_SERVER_ERROR, **e.data)
```

Fortunately, this resource was only used for this single non-conformant contract. Otherwise, I would probably have to split it out to avoid having the custom
validation apply to other endpoints.