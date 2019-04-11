
# Introduction

The System for Cross-domain Identity Management (SCIM) defines an API
that can be used by Identity Providers to automatically provision,
update and deprovision users. TeamViewer implements this API to a
certain extent and therefore allows automatic creation, updates and
deactivation of users in a TeamViewer company.

This article describes the TeamViewer SCIM API and uses the terms and
numbers defined in the ["TeamViewer API Documentation"][tvapidocs]
(chapter 2 - "Introduction").

# Authorization

The TeamViewer SCIM API uses OAuth 2.0 based authorization, as described in the
["TeamViewer API Documentation"][tvapidocs].

The access token can be created in the TeamViewer Management Console and
needs to be given in the HTTP Authorization header, sent by the Identity
Provider. Please see the respective documentation of the IdP on how to
configure the authorization token.

The access token requires to have the following scopes:

* `Users.CreateUsers` (or `Users.CreateAdministrators`)
* `Users.Read`
* `Users.ModifyUsers`
* `Users.ModifyAdministrators` (optional)

This corresponds to the following entries in the TeamViewer Management
Console "Script Token" dialog:

* UserManagement: View, create and edit users
* UserManagement: View, create and edit users and admins (optional)

# SCIM API User Resource

## User Attributes

The following table summarizes the TeamViewer User properties and how
they are mapped to the respective SCIM attributes.

<table>
    <thead>
        <tr>
            <th>TeamViewer User Property</th>
            <th>SCIM Attribute(s)</th>
            <th>Description</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>id</td>
            <td>id</td>
            <td>
                The user ID that is needed to uniquely identify the
                TeamViewer user. <br/>
                This ID needs to start with a "u".
            </td>
        </tr>
        <tr>
            <td>name</td>
            <td>
                displayName <br/>
                name.formatted <br/>
                name.givenName <br/>
                name.familyName <br/>
            </td>
            <td>
                The name of the TeamViewer user. <br/>
                The "displayName" attribute holds the actual name of the
                user when returned by the API. It also has precedence
                over any other name attribute when setting the user
                name.<br/>
                The "name.formatted" attribute holds the same value as
                the "displayName" attribute when returned by the API.
                On input, it is used if the "displayName" is not given.
                <br/>
                The "name.givenName" attribute holds the first part of
                the TeamViewer user name when split by space - the
                "name.familyName" holds the rest. If the name does not
                contain spaces, both values will be null. <br/>
                On input, these values are only considered for the user
                name if neither the "displayName" or "name.formatted"
                attributes are set. The name is then built by separating
                "givenName" and "familyName" with a space character.
            </td>
        </tr>
        <tr>
            <td>email</td>
            <td>
                userName <br/>
                emails[0].value
            </td>
            <td>
                The email address of the TeamViewer user. <br/>
                When returned by the API, the "userName" attribute holds
                the email address of the respective user. <br/>
                TeamViewer supports only a single email address per
                user.
            </td>
        </tr>
        <tr>
            <td>active</td>
            <td>active</td>
            <td>
                The state of the user account. True, if the account is
                enabled, false otherwise.
            </td>
        </tr>
        <tr>
            <td>password</td>
            <td>password</td>
            <td>
                The password for a newly created TeamViewer user. <br/>
                This attribute will never be included in an API response
                and is only used for creating users.
            </td>
        </tr>
        <tr>
            <td>language</td>
            <td>preferredLanguage</td>
            <td>
                The language code for a newly created TeamViewer user.
                This will be used for the welcome email. <br/>
                This attribute will not be included in API responses and
                is only used for creating users.
            </td>
        </tr>
    </tbody>
</table>


## Get a list of users

```
GET /scim/v2/Users
```

Lists all users in a TeamViewer company. The list can optionally be
filtered and paginated.

### Parameters

* **filter** (optional)

    Enables filtering of the user list.

    Example: `filter=userName eq "johndoe@testing.local"`

    * Supported filter operators (matches case-sensitive):

        * `eq` : Attribute must match exactly.
        * `ne` : Attribute must not match.
        * `co` : Attribute must contain given value.
        * `sw` : Attribute must start with given value.
        * `ew` : Attribute must end with given value.

    * Supported attributes to filter on:

        * `userName` (corresponds to email address)
        * `emails.value`
        * `name`
        * `displayName`

* **startIndex** (optional)

    Used for pagination. This is the (1-based) index to start returning
    values for. It defaults to 1 if not present.

* **count** (optional)

    The maximum number of items to include in the result set.

    If not given, all items will be returned, beginning from the
    optionally given "startIndex".

### Example

#### Request
```
GET /scim/v2/Users?filter=userName ew "example.test"
```

#### Response - 200 OK
```json
{
    "schemas": [
        "urn:ietf:params:scim:api:messages:2.0:ListResponse"
    ],
    "totalResults": 3,
    "startIndex": 0,
    "itemsPerPage": 3,
    "Resources": [
        {
            "id": "u1234567",
            "name": {
                "givenName": "John",
                "familyName": "Doe",
                "formatted": "John Doe"
            },
            "displayName": "John Doe",
            "emails": [
                {
                    "primary": true,
                    "value": "johndoe@example.test"
                }
            ],
            "userName": "johndoe@example.test",
            "active": true
        },
        { … },
        { … }
    ]
}
```


## Get a single user

```
GET /scim/v2/Users/{id}
```

Returns the information for a single user. The "id" placeholder must be
replaced by the user's ID as returned by the list-operation.

### Parameters

* **id** (path)

    The user's ID as returned by the list-operation.

### Example

#### Request
```
GET /scim/v2/Users/u1234567
```

#### Response - 200 OK
```json
{
    "schemas": [
        "urn:ietf:params:scim:schemas:core:2.0:User"
    ],
    "id": "u1234567",
    "name": {
        "givenName": "John",
        "familyName": "Doe",
        "formatted": "John Doe"
    },
    "displayName": "John Doe",
    "emails": [
        {
            "primary": true,
            "value": "johndoe@example.test"
        }
    ],
    "userName": "johndoe@example.test",
    "active": true
}
```


## Create a new user

```
POST /scim/v2/Users
```

Creates a new user for the TeamViewer company. The new user is returned
as response to this operation. Optionally, users can be created to be
enabled for Single Sign-On and will therefore not require a TeamViewer
account password.

### Parameters

Given as JSON formatted payload in the request body.

* **schemas**

    Must include the "urn:ietf:params:scim:schemas:core:2.0:User"
    schema.

* **userName**

    The email address for the new user. Will be used for login.

    This attribute has precedence over the "emails" parameter.

* **displayName**

    The name of the new user.

    This attribute has precedence over the "name" parameter.

* **emails**

    List of email addresses of the user. Values will only be evaluated
    if the email address is not given in the "userName" attribute. If
    multiple values are present, the first entry marked as primary email
    address is taken. If no primary email address is given, the first
    entry in the list is considered as the email address of the new
    user.

* **name**

    Parts of the name of the new user.

    The following order applies in building the name for the new user:

    * If given, use the "displayName"
    * Else, if given use the "name.formatted"
    * Otherwise build the name by separating "name.givenName" and
      "name.familyName" by a space-character.

* **password** (optional)

    Predefined password for the user. Will be used for login.

* **preferredLanguage** (optional)

    Language for the new user. Will be used for the welcome email.
    Defaults to "en".

* **ssoCustomerId** (optional)

    The customer identifier that is needed for Single Sign-On. If this
    parameter is specified, newly created users are SSO-enabled and do
    not need a TeamViewer account password.

    This parameter is part of the SCIM extension schema `urn:ietf:params:scim:schemas:extension:teamviewer:1.0:SsoUser`

### Example #1 - Normal User

#### Request
```json
POST /scim/v2/Users
Content-Type: application/scim+json

{
    "schemas": [
        "urn:ietf:params:scim:schemas:core:2.0:User"
    ],
    "userName": "jane.doe@example.test",
    "displayName": "Jane Doe",
    "emails": [
        {
            "value": "jane.doe@example.test",
            "primary": true
        }
    ],
    "name": {
        "givenName": "Jane",
        "familyName": "Doe",
        "formatted": "Jane Doe"
    },
    "password": "secret1!",
    "preferredLanguage": "de_DE"
}
```

#### Response - 201 Created
```json
{
    "schemas": [
        "urn:ietf:params:scim:schemas:core:2.0:User"
    ],
    "id": "u1234567",
    "name": {
        "givenName": "Jane",
        "familyName": "Doe",
        "formatted": "Jane Doe"
    },
    "displayName": "Jane Doe",
    "emails": [
        {
            "primary": true,
            "value": "jane.doe@example.test"
        }
    ],
    "userName": "jane.doe@example.test",
    "active": true
}
```

### Example #2 - Single Sign-On User

#### Request
```json
POST /scim/v2/Users
Content-Type: application/scim+json

{
    "schemas": [
        "urn:ietf:params:scim:schemas:core:2.0:User",
        "urn:ietf:params:scim:schemas:extension:teamviewer:1.0:SsoUser"
    ],
    "userName": "jane.doe@example.test",
    "displayName": "Jane Doe",
    "emails": [
        {
            "value": "jane.doe@example.test",
            "primary": true
        }
    ],
    "name": {
        "givenName": "Jane",
        "familyName": "Doe",
        "formatted": "Jane Doe"
    },
    "preferredLanguage": "de_DE",
    "urn:ietf:params:scim:schemas:extension:teamviewer:1.0:SsoUser": {
        "ssoCustomerId": "a1b2bc4d5e6f7"
    }
}
```

The response is similar to the one given in the first example for this
endpoint.


## Update an existing user

```
PUT /scim/v2/Users/{id}
```

Changes information for the selected user. The "id" placeholder must be
replaced by the user's ID as returned by the list-operation. The "PUT"
operation intends to replace an existing resource. Therefore the request
should include the complete (updated) user information. This includes
attributes that do not change. Responds with the updated User resource.

### Parameters

* **id** (path)

    The user's ID as returned by the list-operation.

Given as JSON formatted payload in the request body:

* **schemas**

    Must include the "urn:ietf:params:scim:schemas:core:2.0:User"
    schema.

* **userName**

    The email address of the user.

    This attribute has precedence over the "emails" parameter.

    Changes the email address of the user if different than the current
    email address. A verification Email will be sent to the new address.

* **displayName**

    The name of the new user.

    This attribute has precedence over the "name" parameter.

    Changes the name of the user if different than the current name.

* **emails**

    List of email addresses of the user. Values will only be evaluated
    if the email address is not given in the "userName" attribute. If
    multiple values are present, the first entry marked as primary email
    address is taken. If no primary email address is given, the first
    entry in the list is considered as the email address of the new
    user.

* **name**

    Parts of the name of the user.

    The following order applies in building the name for the new user:

    * If given, use the "displayName"
    * Else, if given use the "name.formatted"
    * Otherwise build the name by separating "name.givenName" and
      "name.familyName" by a space-character.

* **active**

    The state of the user account. True, if the account is enabled,
    false otherwise.

    Can be used to activate/deactivate the user account.

### Example

#### Request
```json
PUT /scim/v2/Users/u1234567
Content-Type: application/scim+json

{
    "schemas": [
        "urn:ietf:params:scim:schemas:core:2.0:User"
    ],
    "userName": "jane.doe@example.test",
    "displayName": "Jane Doe (Updated Name)",
    "emails": [
        {
            "value": "jane.doe@example.test",
            "primary": true
        }
    ],
    "name": {
        "givenName": "Jane",
        "familyName": "Doe (Updated Name)",
        "formatted": "Jane Doe (Updated Name)"
    },
    "active": true
}
```

#### Response - 200 OK
```json
{
    "schemas": [
        "urn:ietf:params:scim:schemas:core:2.0:User"
    ],
    "id": "u1234567",
    "name": {
        "givenName": "Jane",
        "familyName": "Doe (Updated Name)",
        "formatted": "Jane Doe (Updated Name)"
    },
    "displayName": "Jane Doe (Updated Name)",
    "emails": [
        {
            "primary": true,
            "value": "jane.doe@example.test"
        }
    ],
    "userName": "jane.doe@example.test",
    "active": true
}
```


## Update properties of an existing user

```
PATCH /scim/v2/Users/{id}
```

Changes information for the selected user. The "id" placeholder must be
replaced by the user's ID as returned by the list-operation. The "PATCH"
operation can be used to change single properties of a user. Only the
parts that need to be changed are needed in the request body. Responds
with the updated User resource.

### Parameters

* **id** (path)

    The user's ID as returned by the list-operation.

Given as JSON formatted payload in the request body:

* **schemas**

    Must include the "urn:ietf:params:scim:api:messages:2.0:PatchOp"
    schema.

* **Operations**

    List of operations to apply to the selected user.

    * **op**

        Type of change operation to perform. The only supported
        operation type is "replace".

    * **path** (optional)

        Optional path of the attribute to change. Child attributes of
        need to be separated by dot (.) from their parent attribute
        (e.g. "name.givenName").

    * **value**

        Parts of the user to change.

        If the "path" attribute is omitted, this needs to be a partial
        User object. Otherwise the value of the respective attribute,
        addressed by "path".

        Changing the following attributes is supported:

        * `userName`
        * `displayName`
        * `emails.value`
        * `name.formatted`
        * `name.givenName`
        * `name.familyName`
        * `active`

### Example

#### Request
```json
PATCH /scim/v2/Users/u1234567
Content-Type: application/scim+json

{
    "schemas": [
        "urn:ietf:params:scim:schemas:core:2.0:PatchOp"
    ],
    "Operations": [
        {
            "op": "replace",
            "value": {
                "active": false
            }
        },
        {
            "op": "replace",
            "path": "displayName",
            "value": "Jane Doe Updated"
        }
    ]
}
```

#### Response - 200 OK
```json
{
    "schemas": [
        "urn:ietf:params:scim:schemas:core:2.0:User"
    ],
    "id": "u1234567",
    "name": {
        "givenName": "Jane",
        "familyName": "Doe Updated",
        "formatted": "Jane Doe Updated"
    },
    "displayName": "Jane Doe Updated",
    "emails": [
        {
            "primary": true,
            "value": "jane.doe@example.test"
        }
    ],
    "userName": "jane.doe@example.test",
    "active": false
}
```


[tvapidocs]: https://dl.tvcdn.de/integrate/TeamViewer_API_Documentation.pdf
