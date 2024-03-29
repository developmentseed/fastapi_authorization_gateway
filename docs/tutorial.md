# Tutorial

## Introduction
This library provides a model for defining route-based permission policies for a FastAPI app and a mechanism for evaluating incoming requests against those policies. It concerns itself with Authorization and is independent of any Authentication considerations. Similarly, it does not concern itself with the way policies are generated. That is left up to you, dear user.

This library is most useful in scenarios where you do not have the ability to modify a set of routes, but do want to control access to them. It was created specifically to support authorization on [stac-fastapi](https://github.com/stac-utils/stac-fastapi), but should be generally useful in similar situations.

## Setup

Let's look at an example:

```python
from fastapi import Depends, Request
from fastapi_authorization_gateway.auth import build_authorization_dependency
from fastapi_authorization_gateway.types import Policy, RoutePermission

async def get_user(request: Request):
    return {
        "username": "test"
    }


async def policy_generator(request: Request, user: Annotated[dict, Depends(get_user)]) -> Policy:
    """
    Define your policies here based on the requesting user or, really,
    whatever you like. This function will be injected as a dependency
    into the authorization dependency and must return a Policy.
    """

    # We will generate some policies that cover all routes for the app,
    # so we need to enumerate them here.
    all_routes: list[APIRoute] = request.app.routes

    # A permission matching write access to all routes, with no constraints
    # on path or query parameters
    all_write = RoutePermission(
        paths=[route.path_format for route in all_routes],
        methods=["POST", "PUT", "PATCH", "DELETE"],
    )

    # a permission matching read access to all routes, with no constraints
    # on path or query parameters
    all_read = RoutePermission(
        paths=[route.path_format for route in all_routes], methods=["GET"]
    )

    # read only policy allows read requests on all routes and denies write requests
    # falling back to denying a request if it matches none of the permissions
    read_only_policy = Policy(allow=[all_read], deny=[all_write], default_deny=True)

    # a more permissive policy granting write and read access on all routes, falling back
    # to approving a request if it matches none of the permissions
    authorized_policy = Policy(allow=[all_write, all_read], default_deny=False)
    
    if not user:
        # anonymous requests get read only permissions
        return read_only_policy
    else:
        # authenticated requests get full permissions
        return authorized_policy


# build the authorization dependency
authorization = build_authorization_dependency(
    policy_generator=policy_generator,
)
```

Ok, there are a few things going on here. We have defined two async functions:

`get_user`
This is a stand-in for a function you might actually use to retrieve user details from a request. For example, you might leverage HTTP Basic Auth to validate a username/password against a user database and return the row for the requesting user. Alternatively, you might pull user data out of a token. The details are up to you. You don't even need this function, at all, if it doesn't suit your use case, but we provide it as an example to illustrate how you might inject user data into the `policy_generator` function.

`policy_generator`
This coroutine outputs a permissions Policy that our library can use to evaluate and allow/deny incoming requests. How you define your policies is entirely up to you. For example, maybe you have a user object with a `groups` property and want to assign a more permissive policy to users in the "admin" group than to users in the "viewer" group. Maybe you want to leverage JWT claims or scopes to define your policies. The details are up to you. So long as this function returns a `Policy`, it will work with the authorization library. This function is injected as a dependency on the `authorization` dependency, so it can leverage any FastAPI Dependencies you like, as illustrated here with the `request` and `get_user` dependencies.

Once the `policy_generator` function is defined, we pass it to `build_authorization_dependency`. This returns an authorization function that can be injected into routes, routers or an entire FastAPI app as a dependency. It will evaluate any incoming requests, against a `Policy` generated using the `policy_generator` function.

## Defining Policies

Policies provide a means of evaluating incoming requests against a combination of route path, request method, path parameters and query parameters. It does so using a set of `RoutePermission` instances. Each Policy can define a set of `RoutePermissions` to `allow` and/or `deny`. We will talk about how those permissions are evaluated soon, but first let's look at an example:

```python
get_index = RoutePermission(paths=["/"], methods=["GET"])

Policy(allow=[get_index])
```

In the example above, we create a `RoutePermission` matching GET requests to the root path ("/"). Since we set this permission on the `allow` list for the Policy, any GET requests to "/" will be allowed.

That basic example illustrates the concept, but you can imagine it would be incredibly tedious to define a permission covering every route in your application by hand. Thankfully we don't have to.

### Dynamically Generating Policies

Since our Policies are generated in a dependency, we have access to a Request object (and any other FastAPI dependencies we might like). This allows us the ability to enumerate all available routes and dynamically generate Policies for them.

For example, suppose we have a REST API serving a `Collection` model at `/collections` and `/collectons/{collection_id}` and we allow only admin users to create or update collections. We might enumerate the collection routes like so:

```python
from fastapi import Request
from fastapi_authorization_gateway.types import RoutePermission

def generate_policy(request: Request, user: Annotated[dict, Depends(get_user)]):
    all_routes = request.app.routes
    collection_routes = [route for route in all_routes if route.path_format.startswith("/collections")]
    write_collections = RoutePermission(paths=collection_routes, methods=["PUT", "PATCH", "POST"])

    if user.is_admin:  # is_admin is an imaginary property--defining this is left up to the implementor
        return Policy(allow=[write_collections])
    else:
        return Policy(deny=[write_collections])
```

### Path Parameters

The example above allows us to control access to specific routes, but what if we want to control access to those routes conditionally, depending on which data is being accessed? In other words, we might want to enforce object-level permissions. We can do that using `path_parameters`.

```python
from fastapi import Request
from fastapi.params import Path
from fastapi_authorization_gateway.types import RoutePermission

def generate_policy(request: Request, Annotated[dict, Depends(get_user)]):
    all_routes = request.app.routes
    collection_routes = [route for route in all_routes if route.path_format.startswith("/collections")]
    user_collection_ids = ["collectionA", "collectionB"]  # these could come from anywhere, including a database, token, API, whatever
    user_collection_regex = r"^(" + "|".join([re.escape(c) for c in user_collection_ids]) + r")$"
    write_user_collections = RoutePermission(
        paths=collection_routes,
        methods=["PUT", "PATCH", "POST"],
        path_params={"collecton_id": Annotated[str, Path(pattern=user_collection_regex)]}
    )
    return Policy(allow=[write_user_collections])
```

Note that we leverage FastAPIs `Path` class to define the path parameter conditions. Doing so allows us to make use of the full matching capabilities of that class. We use `pattern` here to match the path parameter against a regular expression, but we could also use something like `lt` or `gt` to match against a range of numeric values.

### Query Parameters

We may also want to control access depending on which query parameters are specified on a request. We can do so in a similar fashion to the Path Parameters:

```python
from typing import Annotated
from fastapi import Request
from fastapi.params import Query
from fastapi_authorization_gateway.types import RoutePermission

def generate_policy(request: Request, Annotated[dict, Depends(get_user)]):
    all_routes = request.app.routes
    if user.is_authenticated:
        max_count = 100
    else:
        max_count = 10
    read_collections = RoutePermission(
        paths=["/collections"],
        methods=["GET"],
        query_params={"limit": Annotated[int, Query(le=max_count)]}
    )
    return Policy(allow=[read_collections], default_deny=True)
```

In the example above, we allow authenticated users to request a higher limit from the Collections list route than anonymous users. It's an arbitrary example, but illustrates how you might control requests using a numerical value in a query parameter.

Note the addition of `default_deny=True` in the Policy. We haven't seen this yet, but it's a flag we can set which determines whether a request which matches none of the defined permissions is allowed or denied. For example, in this case, if an anonymous user makes the following request: GET `/collections?limit=11` it will not match the Allow permission set on the Policy, which limits `limit` to 10 or less. It will therefore fallback to the default condition, which we have now set to deny. If we wanted to be explicit about this constraint, we could invert the query param condition and add it to a deny policy. For example:

```python
from fastapi import Request
from fastapi.params import Query
from fastapi_authorization_gateway.types import RoutePermission

def generate_policy(request: Request, Annotated[dict, Depends(get_user)]):
    all_routes = request.app.routes
    if user.is_authenticated:
        max_count = 100
    else:
        max_count = 10
    read_collections_le = RoutePermission(
        paths=["/collections"],
        methods=["GET"],
        query_params={"limit": Annotated[int, Query(le=max_count)]}
    )
    read_collections_gt = RoutePermission(
        paths=["/collections"],
        methods=["GET"],
        query_params={"limit": Annotated[int, Query(gt=max_count)]}
    )

    return Policy(allow=[read_collections_le], deny=[read_collections_gt])
```

## Policy Evaluation

This bring us to how Policies are evaluated. A Policy will prioritize the deny list over the allow list and fallback to the `default_deny` setting if a request does not match any RoutePermissions defined in the deny or allow lists. The evaluation logic looks like this:

1. Does the incoming request match any RoutePermissions in the `deny` list? If so, deny the request.
2. Does the incoming request match any RoutePermissions in the `allow` list? If so, allow the request.
3. Is `default_deny` set to True? If so, deny the request.
4. If none of the above conditions are met, allow the request.


## Request Transformation

Authorization is not only about allowing or denying access to a route. In some cases, it makes sense to mutate a request in order to only return data that the user is allowed to access. For example, we may want to filter queries passed to a Search endpoint in order to avoid returning unauthorized data. fastapi-authorization-gateway enables this functionality, but it does require that we set up our authorization layer a bit differently.

In order to mutate a request before passing it on to the underlying endpoint, we wrap all endpoints in a generic receiving function, which runs the usual authorization dependency and then executes any desired request transformations prior to passing everything on to the original endpoing. In order to accomplish this, we need to re-register all endpoints on the router, replacing them with their wrapped counterparts. In practice, what this means for you is the code to add fastapi-authorization-gateway to your app will change from this:

```python
class SearchBody(BaseModel):
    collections: list[str] = Field(default_factory=list)


def transform_search(
    request: Request, policy: Policy, search_body: SearchBody, *args, **kwargs
):
    """
    Filter the requested collections to only those that the user has access to.
    """
    search_body.collections = [
        collection
        for collection in search_body.collections
        if collection in policy.metadata["collections"]
    ]


async def policy_generator(
    request: Request, user: Annotated[dict, Depends(get_user)]
) -> Policy:
    """
    Return a Policy allowing POST requests to /search, but
    modifying incoming requests to restrict access to specific
    collections.
    """
    search = RoutePermission(
        paths=["/search"],
        methods=["POST"],
    )

    request_transformations = [
        RequestTransformation(
            path_formats=["/search"],
            transform=transform_search,
        )
    ]

    policy = Policy(
        allow=[search],
        request_transformations=request_transformations,
        metadata={"collections": user["collections"]},
    )

    return policy

authorization = build_authorization_dependency(
    policy_generator=policy_generator,
)

app = FastAPI(dependencies=[Depends(authorization)])
```

to this:

```python
from fastapi_authorization_gateway.auth import wrap_router
authorization = build_authorization_dependency(
    policy_generator=policy_generator,
)

app = FastAPI()

# define all routes
@app.post("/search")
def search(request: Request, search_body: SearchBody):
    return search_body

# after defining all routes, we can wrap the router, replacing
# all routes with wrapped versions
wrap_router(app.router, authorization_dependency=authorization)
```

### RequestTransformation

A `RequestTransformation` object determines how specific routes will have their incoming request data transformed. They are pretty simple in structure: we define a list of `path_formats` which will be matched against the `path_format` of the Route for a request. If they match, the `transform` function will be passed the `Request`, the active `Policy` and any other arguments defined on the original endpoint.

### Transform functions

A transform function should accept a `Request` object, a `Policy` and any other parameters passed to the original endpoint, if they are useful for the transformation. The transform function returns nothing. Any mutations should be done in-place, since the objects being modified are later passed on to the original endpoint. In the example above, our transform function accepts the `search_body` parameter and modifieds its `collections` property in-place.

