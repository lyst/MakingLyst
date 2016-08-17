Tooling at Lyst
---------------

At Lyst we use Python. This extension document focuses on tools available to Python that can and should be used for writing HTTP API services. It's intended for all Python engineers.

We've kept this separate from the main document so it stays language agnostic.

## Contents

- [Frameworks](#frameworks)
- [Testing](#testing)
- [Monitoring](#monitoring)
- [Logging](#logging)
- [Documenting](#documenting)

## Frameworks

There are many frameworks available to Python for HTTP APIs. Too many, in fact. It can make it hard to pick the best one.

At Lyst we use [django](https://djangoproject.com), a _"batteries included"_ webframework that handles user authentication, database interaction, html template rendering and much much more. Considering this, it's a good idea to use tools that work well with Django.

[Django REST Framework](http://django-rest-framework.org) is the obvious choice here, and we use it extensively. We even use components like [serializers](http://www.django-rest-framework.org/api-guide/serializers/) for representing components in Lyst's UI, not just for data representations. Try to get as familiar with Django REST Framework as you are with Django.

We recommend you read the [getting started guide](http://www.django-rest-framework.org/tutorial/quickstart/) to get familiar with the concepts.

If you are less familiar with Django, [Swagger](http://swagger.io) has a plugin for Django that makes it much easier to build basic CRUD-style HTTP API services with Django. The configuration is not as complicated or as versatile as Django REST Framework. However you can get something up and running much faster. Unless you absolutely cannot use Django REST Framework for lack of technical knowledge, swagger is a good alternative.

## Testing

For testing, we prefer integration tests that perform repeated HTTP requests to endpoints with slightly different parameters. This allows us to test the edgecase behaviour of those endpoints in a way we expect clients to use the API service.

We use [pytest](http://pytest.org/latest/) extensively for integration testing. Here is an example from our codebase that tests the user endpoint functionality:

```python

@pytest.mark.django_db
@pytest.mark.parametrize('base_url, response_status, use_slug', [
    ('/api/users/{}/', status.HTTP_200_OK, True),
    ('/api/users/i-do-not-exist/', status.HTTP_404_NOT_FOUND, False),
])
def test_user_get(http_client, base_url, response_status, use_slug):
    """
    Test getting users that exist and do not exist result in the correct responses.
    """
    if use_slug:
        user = UserFactory()
        base_url = base_url.format(user.slug)
    response = http_client.authorized_get(base_url)
    assert response.status_code == response_status
```

We're using a fixture for the `http_client` attribute. This is an http client with an authenticated user for us, so we don't need to authenticate each time. We'd use a different client for an anonymous user.

In our API tests, we use the [parametrize](http://pytest.org/latest/parametrize.html) mark to run through the test each time with slightly different conditions, in this case: a different url and status code.

This test only checks the response status from the service but suffices for the purpose of this test. A separate test that inspects the response body attributes would also help to ensure this endpoint is behaving as expected and well covered.

[Read more about pytest.](http://pytest.org/latest/)

## Monitoring

Monitoring of your live API service behaviour is important. Once an API service is alive and in the wild, it needs to have an acceptable response time that is consistent.

At Lyst we use [Runscope](https://runscope.com) for monitoring our API services. Runscope acts like a client of your API service. It can perform HTTP requests and inspect the responses. You can define success criteria by inspecting the body of the response or the response time. You can also perform the tests from many locations throughout the world. This can help to ensure that your API service performs consistently across the globe.

At Lyst, we aim to make sure all API response times are below 500ms on average. This is about the same level that is accepted on good websites. Alongside this, we also check _bad responses_ too. We want to make sure the live API errors in the same way as it does in testing.


If you are interested in discovering bottlenecks in your internal services (your software, your caching or your database access), then a tool like [New Relic](https://newrelic.com) is better suited for the task.

## Logging

At Lyst we use [Sentry](https://www.getsentry.com/welcome/) to log all errors that occur in our API service. A good practice is to log each expected error as a **warning** and unexpected behaviour as an **exception**. It's likely that your clients will send bad requests as they integrate your service. You do not need to flag severe warnings for these. Differentiating between the severity level will also make it easier to filter the logs.

A **warning** could be a user sending bad request parameters. An **exception** could be an unexpected server error.

## Documenting

> _A great API service with no documentation is a shit API service._

Documentation for API services can take two paths:

1. Autogenerated documentation
2. Manually written documentation

There is no best option and it is dependent on the type of API service you are providing and the knowledge of your clients.

### Autogenerated documentation

If you are building a simple CRUD-style API service then autogenerated documentation is definitely a path you should consider.

[Sphinx](http://sphinx-doc.org) is a defacto-standard for autogenerating documentation in Python. Sphinx has a contribution package called [sphinxcontrib.httpdomain](https://pythonhosted.org/sphinxcontrib-httpdomain/) specifically for marking up HTTP API documentation. Here is an example:

```rst
.. http:get:: /users/(string:user_uuid)

   A User representation

   **Example request**:

   .. sourcecode:: http

      GET /users/9a31bb89-5954-4843-aabd-783ceaf410c6 HTTP/1.1
      Host: myapi.com
      Accept: application/json
      Authorization: Bearer 1af2b049-1043-4fc7-a774-9ba16d0e1623

   **Example response**:

   .. sourcecode:: http

      HTTP/1.1 200 OK
      Vary: Accept
      Content-Type: application/json

        {
          "username": "Tyler Durden",
          "email": "tyler@paperstreetsoap.com"
        }

   :reqheader Accept: the response content type depends on
                      :mailheader:`Accept` header
   :reqheader Authorization: required OAuth token to authenticate
   :statuscode 200: User representation returned successfully
   :statuscode 404: User with uuid does not exist
```

If this is written as docstrings in Python, you can use [autodoc](http://sphinx-doc.org/ext/autodoc.html) and sphinx can turn this into documentation for you.

### Manually written documentation

If you are building an API service that provides domain-specific functionality then writing the documentation by hand is the prefered option. Automatic documentation can be ignorant of the features of your service even if the HTTP component looks similar to a CRUD-style interface.

For API services that are products; documentation is part of that product. You should give special attention to the experience it provides your clients. This is one of the reasons why handwritten documentation is prefered.

For statically hosted documentation [mkdocs](https://mkdocs.org) is a fantastic tool. Mkdocs is markdown-based and is easy to use. It provides GitHub pages integration but it can also generate static html files. The flexibility for custom themes also means you can brand it to match the rest of your services.
