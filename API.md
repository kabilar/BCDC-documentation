# BCDC Ingest API

This document will describe the objects and actions associated with the BCDC Ingest API.

The types of resources contained in BCDC will include users, namespaces, schemas, and objects.
Users will be associated with one or more namespaces. A namespace can contain multiple objects.
Schemas are not namespaced and exist globally. Each object must be validated with a schema.
There exists a global namespace that contains objects that are globally available for linking.

## Users

As stated above, users will be associated with one or more namespaces. By being associated with
a namespace, users will have the following permissions.

Within their namespace, users will be able to
* read and write objects
* link to objects

For objects in other namespaces, users will be able to
* read objects
* link to objects in other namespaces

API users are not special. They are represented as regular users.
API users may have an attribute associated with the user to signify that their primary purpose is for API access.

## Roles

Along with these default user permissions, users may be associated with one or more roles.
Each role is associated with a set of permissions.

### API role

There is no need for a special API role. This can be represented as a regular user.

### Administrator role

Administrators will be able to create and edit objects in any namespace.

### Schema Editor role

Schema editors will be able to create and edit schemas.

### Namespace Creator role

Namespace creators will be able to create namespaces.

### User Administrator role

User administrators will be able to create users.
User administrators will be able to associate users to namespaces.

### Super User role

The super user role is the superset of all roles and contains all permissions.

## Namespaces

A namespace is a sandboxed collection of objects.
Each namespace will be globally unique and will be assigned by a BCDC namespace creator.
Each set of projects will be assigned a namespace.
Objects associated with the project will be contained within the namespace.

## Schemas

A schema is a JSON Schema used to describe and validate an object.
Schemas are versioned.
Each schema has an identifier (`name`) that is unique.
Objects must be have a type that is defined by one of these schemas (specified by schema identifier).

## Objects

An object is a data structure that exists within a namespace.
Objects are versioned.
Each object can be identified by the tuple of the following properties: `namespace`, `type`, and `name`.
Each object has a property (`schema`) that corresponds with the JSON schema that the object has been validated with.
This `schema` property has the component parts of `name` and `version` that corresponds with the
schema name, and version that identifies the specific JSON schema document.
The `type` property is an arbitrary grouping of objects.
All objects with the same value of `type` should share the same `schema` values (`name`, `version`),
but this is not an absolute requirement.
The `name` property must be unique within the set of objects that share the same `namespace` and `type` values.

## Object linking

Objects may be linked to other objects, even objects in other namespaces.
This linking will be indicated in the schema by a custom JSON schema keyword: `foreignKey`.

In this contrived example, the example object conforms to the schema.
The `derived_from` property references objects in the `generic` namespace of type `donor`
with the names `foo donor` and `bar donor`.
If the referenced object does not already exist, this example object will not pass validation.

ex.

    schema = {
      "type": "object",
      "properties": {
        "name": { "type": "string" },
        "derived_from": {
          "type": "array",
          "items": {
            "type": "string"
            "foreignKey": {
              "namespace": "generic",
              "type": "donor"
            }
          }
        }
      },
      required: [ "name", "derived_from" ]
    }

    object = { "name": "example object", "derived_from": [ "foo donor", "bar donor" ] }

## Object Publishing

Objects have various states (meta-properties) associated with them.

Objects have an `approved` state. The value may be `true` or `false`.

Objects may have `deleted` state. The value may be `true` or `false`.

Objects, once approved, may be marked for publishing.
Objects may have a `marked` state set. The value may be `true` or `false`.

`deleted` and `approved` states are orthogonal to each other.

The `marked` state has some exclusions with the other states.

* only `approved` objects can be marked
* `marked` objects cannot be deleted, they must be unmarked before being deleted
* `marked` objects cannot be unapproved, they must be unmarked before being unapproved

If these excluded state transitions are attempted, the API server should return an error.

It is also an error if one tries to delete an object that is being referenced by another object.

TODO: What to do about publishing objects that reference other objects in different states

## API documentation

### User

#### `PUT /users/token`

Retrieve a new access token. This token, once retrieved, is active for 30 minutes (1800 seconds). The token can be refreshed
in order to extend the session time by another 30 minutes. Once retrieved, this token invalidates all previous
access tokens for this user.

The request payload is JSON object containing the username and password of the requested user.

ex.

    {
        "username":"<username>",
        "password":"<password>"
    }

The response will return a HTTP 200 status if successful.
The response object will also return the access token as well
as other associated information about the session.

ex.

    {
      "status": {
        "success": true,
        "messages": [ "success" ]
      },
      "data": {
        "access_token": "<access token>",
        "created_at": "<created time>",
        "expires_in": 1800,
        "refresh_token": "<refresh token>",
        "token_type": "bearer",
        "username": "<username>"
      }
    }

The response will return a HTTP 401 UNAUTHORIZED status if unsuccessful.

ex.

    {
      "status": {
        "success": false,
        "messages": [ "unauthorized" ]
      },
    }

#### `GET /users/token/<access token>`

Retrieve the requested access token.
This request requires authentication using an access token.
Only the user of the access token or a super-user can retrieve information about an access token.
There is no request payload.

The response will return a HTTP 200 status if successful.
The response object will return the current value for `expires_in`.

ex.

    {
      "status": {
        "success": true,
        "messages": [ "success" ]
      },
      "data": {
        "access_token": "<access token>",
        "created_at": "<created time>",
        "expires_in": 1799,
        "refresh_token": "<refresh token>",
        "token_type": "bearer",
        "username": "<username>"
      }
    }

When unsuccessful, the response will return a HTTP 401 UNAUTHORIZED status.

ex.

    {
      "status": {
        "success": false,
        "messages": [ "unauthorized" ]
      },
    }

#### `PUT /users/token/<access token>`

This API endpoint extends the current sesson for another 30 minutes.
This request requires authentication using an access token.
Only the user of the access token or a super-user can refresh an access token.
The refresh token must be provided in the payload.
The refresh token will change after this endpoint is called.

ex.

    {
        "username":"<username>",
        "refresh_token":"<refresh_token>"
    }


The response will return a HTTP 200 status if successful.
The response object will also return the access token as well
as the new refresh token.

ex.

    {
      "status": {
        "success": true,
        "messages": [ "success" ]
      },
      "data": {
        "access_token": "<access token>",
        "created_at": "<created time>",
        "expires_in": 1800,
        "refresh_token": "<new refresh token>",
        "token_type": "bearer",
        "username": "<username>"
      }
    }

The response will return a HTTP 401 UNAUTHORIZED status if unsuccessful.

ex.

    {
      "status": {
        "success": false,
        "messages": [ "unauthorized" ]
      }
    }

#### `GET /users/<username>`

Retrieve the user with the given username.

#### `PUT /users/<username>`

Create/update the user with the given username.

#### `GET /users/namespaces/<namespace>`

Retrieve users within the given namespace.

#### `GET /users/roles/<role>`

Retrieve users that have the given role.

### Namespace

API methods dealing with namespaces

#### `GET /namespaces`

Retrieve all the namespaces

#### `GET /namespaces/<namespace name>`

Retrieve the namespace named `<namespace name>`.

#### `PUT /namespaces/<namespace name>`

Create/Update the namespace named `<namespace name>`.

### Schemas

#### `GET /schemas`

Retrieve an index of all the schemas.

#### `GET /schemas/<schema name>`

Retrieve the schema named `<schema name>`.

#### `GET /schemas/<schema name>/<version number>`

Retrieve the schema named `<schema name>` with version number `<version number>`.

#### `PUT /schemas/<schema name>`

Create/update the schema identified with name `<schema name>`.
This will create a new schema at version 1 or increment the version number if a schema already exists.

### Objects

#### `GET /objects/<namespace name>`

Retrieve an index of all the object types in the namespace named `<namespace name>`.

#### `GET /objects/<namespace name>/<object type>`

Retrieve an index of all the objects of type `<object type>` in the namespace named `<namespace name>`.

#### `GET /objects/<namespace name>/<object type>/<object id>`

Retrieve the object of type `<object type>` identified by `<object id>` in the namespace named `<namespace name>`.

#### `GET /objects/<namespace name>/<object type>/<object id>/<version number>`

Retrieve the object of type `<object type>` identified by `<object id>` in the namespace named `<namespace name>` with version number `<version number>`.

#### `PUT /objects/<namespace name>/<object type>/<object id>`

Create/update the object of type `<object type>` identified by `<object id>` in namespace `<namespace name>`.
This will create a new object at version 1 or increment the version number if an object already exists. 
Objects must pass schema validation as well as referential integrity of object references to be successfully updated.

#### Object Filters

Given an object query, objects can be filtered by select criteria (mostly date range). TODO: Expand an example.
