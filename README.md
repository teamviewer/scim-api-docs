
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


[tvapidocs]: https://dl.tvcdn.de/integrate/TeamViewer_API_Documentation.pdf
