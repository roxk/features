# auth gear and Cloud Function interaction overview in skygear next

Base on the new product architecture decision (auth gear + Cloud Function and drop record gear), skygear next should bring up a new experience on how auth gear is used in Cloud Function and Client SDK.

## Glossary

* auth gear: a skygear provided component which utilizes user authentication and authorization process, and user object handling. It would bring enhanced features in the future, such as: JWT provider, Auth UIKit, user management dashboard.
* Cloud Function: a developer would create a Cloud Function to fulfill application requirements. A Cloud Function should be a single purpose that attached to certain events or triggered by requirement.
* user auth data: any data that may affect or affected by user authentication status or authorized status, such as disabled, last login at, roles, ..., etc.
* user metadata: any user attributes that are not auth data specified, they include common user attributes, such as avatar, first name, last name, ..., etc. A developer can also store arbitrary form user attributes here.
* user object: merge user auth data and user metadata together. It is in following abstract format:
  ```
  {
      ...auth_data,
      metadata: {
          ...common_user_attributes,
          ...any_other_arbitrary_form_attributes,
      }
  }
  ```

## Goal

* Consider future development of auth gear, including new features such as a User Management Dashboard, UIKit, JWT Provider etc.
* Expected usage is for client-side SDK for logged in user actions, while security-related APIs shall be called via Cloud Functions.
* Avoid coupling between cloud function and auth gear, consider auth gear would be used independently, any access to data owned by another service should only happen through APIs.
* Auth gear and Cloud Function both should have current user execution context.

## Major changes

* Move admin related features from Client SDK to APIs at Cloud Functions.
* API gateway should inject "current user" into request context and then dispatch request to auth gear or Cloud Function.
* Auth gear should extend core user to support user metadata.

## Architecture overview

![](https://i.imgur.com/jxNxhmd.png)

\* this diagram is an overview only, detail flow explain in later section.

- Cloud Functions may have its own DB.
- It's developer's responsibility to create corresponding Cloud Function to handle admin related auth features (enable/disable user, assign role, ...).
- Cloud Function would implement hook functions to handle user changes.

## Move auth admin APIs from SDK to Cloud Function

Following function will be removed from client SDK, a developer should create its Cloud Function to handle admin tasks.

| Attribute | Interface | Old JS SDK API |
| -------- | -------- | ------ |
| disabled | `POST /auth/disable/set`| `adminDisableUser`<br/>`adminEnableUser` |
| password | `POST /auth/reset_password`| `adminResetPassword` |
| role | `POST /auth/role/assign`<br/>`POST /auth/role/admin`<br/>`POST /auth/role/default` | `assignUserRole`<br/>`revokeUserRole`<br/>`setDefaultRole` |

For security reason, Cloud Function should invoke admin related API with master key.

## Execution flow of hooked Cloud Function

There are four hooked Cloud Function forms: 

- `before_XXX_sync`
- `before_XXX`
- `after_XXX_sync`
- `after_XXX`

And as the name inferred, hooked Cloud Functions are executed in two ways: `sync` and `async` way (without `sync` suffix implies `async`). All hooks are executed in transaction. `before_XXX_sync` and `before_XXX` is executed "before" DB operation. Whereas `after_XXX_sync` and `after_XXX` is executed "after" DB operation. `before_XXX_sync` and `after_XXX_sync` can raise exception to abort current operation.

Following pseudo code demonstrates the execution flow in auth gear:

```go=
// start DB transaction
txContext.BeginTx()

err = hooks.ExecuteBeforeSyncHooks(&user)
if err != nil {
    response.Err = skyerr.MakeError(err)
    h.TxContext.RollbackTx()
    return response
}

hooks.ExecuteBeforeAsyncHooks(user)

resp, err := handle(payload, &user)
if err != nil {
    response.Err = skyerr.MakeError(err)
    h.TxContext.RollbackTx()
    return response
}

err = hooks.ExecuteAfterSyncHooks(user)
if err != nil {
    response.Err = skyerr.MakeError(err)
    h.TxContext.RollbackTx()
    return response
}

hooks.ExecuteAfterAsyncHooks(user)

// DB commit
txContext.CommitTx();

response.Result = resp
return response
```

In `before_XXX_sync` and `before_XXX`, it may alter user metadata, on the contrary, `after_XXX_sync` and `after_XXX` won't support to alter user metadata. And all hooks won't support alter user auth data.

|  | alter metadata | alter auth data | raising an exception to stop operation |
| -------- | -------- | -------- | ------ |
| `before_XXX_sync`  | ✓ | 🚫 | ✓ |
| `before_XXX`  | 🚫 | 🚫 | 🚫 |
| `after_XXX_sync`  | 🚫 | 🚫 | ✓ |
| `after_XXX`  | 🚫 | 🚫 | 🚫 |


Function signature of `before_XXX_sync` (`after_XXX_sync` is similar) hooked Cloud Function is:

```javascript
const skygear = require('skygear');

/* 
 * user: user object to be modified
 * context: current exection context
 */
function before_XXX_sync(user, context) {
    console.log(user.metadata.loveCat); // false
    
    // alter user metadata
    user.metadata.loveCat = true;
    
    /*
     * or rasie exception
     * throw new SkygearError("some error");
     */

    // return updated user object
    return user;
}

module.exports = skygear.auth.before_XXX_sync(before_XXX_sync);
```

Function signature of `after_XXX` (`before_XXX` is similar) hooked Cloud Function is:

```javascript
const skygear = require('skygear');

/* 
 * user: user object to be modified
 * context: current exection context
 */
function after_XXX(user, context) {    
    console.log(body); // { "loveCat": false }
    console.log(user.metadata.loveCat); // true
}

module.exports = skygear.auth.after_XXX(after_XXX);
```

Followings are hooks of auth actions:

| Action | Hooked Cloud Function |
| -------- | -------- | 
| `signup` | `before_signup_sync(user, context)`<br/>`before_signup(user, context)`<br/>`after_signup_sync(user, context)`<br/>`after_signup(user, context)`<br/> |
| `login` | `before_login_sync(user, context)`<br/>`before_login(user, context)`<br/>`after_login_sync(user, context)`<br/>`after_login(user, context)` |
| `logout` | `before_logout_sync(user, context)`<br/>`before_logout(user, context)`<br/>`after_logout_sync(user, context)`<br/>`after_logout(user, context)` |
| `roles` | `before_roles_changed_sync(user, context)`<br/>`before_roles_changed(user, context)`<br/>`after_roles_changed_sync(user, context)`<br/>`after_roles_changed(user, context)` |
| `enable` | `before_enable_sync(user, context)`<br/>`before_enable(user, context)`<br/>`after_enable_sync(user, context)`<br/>`after_enable(user, context)`<br/>`before_disable_sync(user, context)`<br/>`before_disable(user, context)`<br/>`after_disable_sync(user, context)`<br/>`after_disable(user, context)` |
| `password` | `before_password_changed_sync(user, context)`<br/>`before_password_changed(user, context)`<br/>`after_password_changed_sync(user, context)`<br/>`after_password_changed(user, context)` |
| `verify` | `before_verified_sync(user, context)`<br/>`before_verified(user, context)`<br/>`after_verified_sync(user, context)`<br/>`after_verified(user, context)`<br/>`before_unverified_sync(user, context)`<br/>`before_unverified(user, context)`<br/>`after_unverified_sync(user, context)`<br/>`after_unverified(user, context)` |
| `metadata` | `before_metadata_modified_sync(user, context)`<br/>`before_metadata_modified(user, context)`<br/>`after_metadata_modified_sync(user, context)`<br/>`after_metadata_modified(user, context)` |
| `user_object` | `blocking_user_sync(user)`<br>`user_sync(user)`<br>(P.S. Naming and implementation detail is not confirmed yet. They will be revisit after launch first version.) |

### Design choices

1. Hooks listed presented here are based on a assumption that a developer could use `content.req.path` to know the reason of certain user auth data changed.

   For example, a user's password changed could be due to:

   1. admin reset a user's password.
   2. a user changes password by himself.
   3. a user forgot password, and reset password via email form.

   They are corresponding to following `context.req.path`:

   1. `/reset_password`
   2. `/change_passowrd`
   3. `/forgot_password/reset_password`

2. `user_sync` and `blocking_user_sync` are special hooks, they are triggered __AFTER__ an user object is added or there is any changes to an user object, they can be used to implement an external user profile store. It also can use as one hook to capture all changes of the user auth data or metadata.

   Note that `user_sync` as an `AFTER`, `ASYNC` hook, its implementaion could be a simple after async hook like other hooks, or it can be optimized by other mechanisms (batching data update), so the `context` arguments are removed.

   `blocking_user_sync` is an `AFTER`, `SYNC` hook. `blocking_user_sync` is especially useful when consistency is important to the application. Skygear next will rollback DB if hook failed.

   _Naming and implementation detail of `user_sync` and `blocking_user_sync` are not confirmed yet. They will be revisit after launch first version._

3. And to avoid spiral request loop, it is forbidden to send request to auth gear in hooked Cloud Function.

4. `context` should contain following information:

   1. `context.user`: user who triggers the hook, ex: client user or admin.
   2. `context.req.path`: original request path from client, ex: for login, it is `/auth/login`.
   3. `context.req.body`: original request body from client, ex: for login, it would be an object with username and password.
   4. `context.req.id`: original request ID.

5. There could be concurrent issue for after hook opersion, consider folowing case:

   | Req 1 | Req 2| Remarks 
   | ---- | ---- | ---- |
   | enable_op |  | Req 1 called |
   | after_enabled_sync() | disable_op | Req 2 called |
   |  | after_disabled_sync() | |
   |  | [external system update user status to disabled] | For network latency reason, req 2's hook got called first |
   | [external system update user status to enabled] | | Oops

   Req 1 and Re1 2 are two requests happens concurrently in race condition, e.g. one users logged in two devices at the same time. This would cause external DB and auth gear data inconsistency.

6. For `user_sync`, `blocking_user_sync` and concurrent hook data inconsistency problem, please refer [#287](https://github.com/SkygearIO/features/issues/287) for more information.
7. Hook execution is "BEFORE" and "AFTER" handler(`handle(payload, &user)`), this design is not the same with skygear v1's record hooks. The reason is because one handler may contain multiple DB operations, so there doesn't exist exactly one DB operation to have before and after hooks. And second the intention is also different from record hooks, record hooks is for before or after DB operation, here is before or after auth operation handles.
8. The `user` parameter is `nil` in `before_signup` hooks.
9. The `user` parameter conveys original `user` object in before hooks. For example, `before_roles_changed_sync(user, context)`, `user` object contains roles before changed.

## user metadata

For future advanced management requirements, auth gear should have user metadata, it includes common user attributes, such as avatar, first name, last name, display name, preferred language, ..., etc. 

Common user attributes would be great help for better auth gear use experience, which allows to provide API response in preferred language,  multi-lang custom email template.

User metadata is also allowed to store varied user attributes, they differ from application to application, such as: ethnic, height, weight, hobby, ..., etc. It is defined by developer freely.

```
CREATE TABLE _auth_user_metadata (
  user_id text REFERENCES _core_user(id),
  created_at timestamp without time zone NOT NULL,
  created_by text,
  updated_at timestamp without time zone NOT NULL,
  
  updated_by text,
  avatar_url text,
  first_name text,
  last_name text,
  display_name text,
  birthday imestamp without time zone,
  gender text,
  prefer_lang_id text REFERENCES _core_lang(id),
  ...
  
  data jsonb,
  
  PRIMARY KEY(user_id),
  UNIQUE (user_id)
);
```

The structure of user object could be:

```
{
    id: <id>,
    created_at: <created_at>,
    updated_at: <updated_at>,
    disabled: <disabled>,
    roles: [<role>, <role>, <role>, ...],
    metadata: {
        avatar_url: <avatar_url>,
        birthday: <birthday>,
        preferred_lang: <preferred_lang>,
        // any other user common attributes
        ...
        // any other arbitrary form attributes
        ...
    }
}
```

## API gateway should add “current user” in request context

Since Cloud Function would indicate it is executed by authenticated user only, so API gateway may handle the authentication process, and generate "current user" context for both auth gear and Cloud Function. (Currently, API gateway didn't handle user authentication process, it is handled by auth gear middleware)

![](https://i.imgur.com/ohlsXTW.png)
1. SDK send Cloud Function request to API gateway.
2. API gateway check if the user is authenticated.
3. If the user is authenticated, API gateway routes the request to Cloud Function.

```javascript=
const skygear = require('skygear');

module.exports = (req, res) => {
    res.end("welcome " + req.context.user.email);
}
```

## Use cases: save/update user in Cloud Function DB

Cloud Function may want to create and sync a user table in their own database, below is an example how to do it:

```javascript
const skygear = require('skygear');
const mongoClient = require('mongodb').MogoClient;

// CF could use after_signup hook to fulfill the requirement
function after_signup_sync(user, context) {
    // connect to Cloud Function DB
    const secrets = context.secrets;
    const mongoUrl = 'mongodb://' + secrets.MONGODB_HOST + ':' + secrets.MONGODB_PORT + '/' + secrets.MONGODB_DBNAME;
    mongoClient.connect(mongoUrl, (err, db) => {
        if (err) {
            throw err;
            return;
        }
        
        // create Cloud Function's user object
        const cloudFunctionUser = {
            id: user.id,
            birthday: user.metadata.birthday,
            avatar: user.metadata.avatarUrl,
            maritalStatus: user.metadata.maritalStatus,
        };
    
        db.collection("users").insertOne(cloudFunctionUser);
    });
}

module.exports = skygear.auth.after_signup_sync(after_signup_sync);
```

Client SDK could use `updateUserMetadata` API to update a user's metadata:


```javascript=
const currentUser = skygear.auth.user;
currentUser.metadata.avatarUrl = "http://example.com/a.jpg";
currentUser.metadata.loveCat = false;
skygear.auth.updateUserMetadata(currentUser).then((currentUser) => {
  console.log(currentUser.metadata.loveCat);
}, (error) => {
  console.error(error);
})
```

To validate changes of user metadata, Cloud Function would hook `before_update_metadata_sync`

```javascript
const skygear = require('skygear');

function before_update_metadata_sync(user, context) {
    if (!user.metadata.loveCat) {
        throw new SkygearError("EVERYONE LOVES CAT");
    }

    return user;
}

module.exports = skygear.auth.before_update_metadata_sync(before_update_metadata_sync);
```

And then Cloud Function saved updated user

```javascript=
const skygear = require('skygear');
const mongoClient = require('mongodb').MogoClient;

// CF could use after_signup hook to fulfill the requirement
function after_update_metadata_sync(user, context) {
    // connect to Cloud Function DB
    const secrets = context.secrets;
    const mongoUrl = 'mongodb://' + secrets.MONGODB_HOST + ':' + secrets.MONGODB_PORT + '/' + secrets.MONGODB_DBNAME;
    mongoClient.connect(mongoUrl, (err, db) => {
        if (err) {
            throw err;
            return;
        }
        
        // create Cloud Function's user object
        const cloudFunctionUser = {
            id: user.id,
            birthday: user.metadata.birthday,
            avatar: user.metadata.avatarUrl,
            maritalStatus: user.metadata.maritalStatus,
        };
    
        db.collection("users").update({id: user.id}, {$set: cloudFunctionUser});
    });
}

module.exports = skygear.auth.after_update_metadata_sync(after_update_metadata_sync);
```

## Use case: auth gear as JWT provider

Since auth gear has the knowledge of user metadata, it could generate authenticated JWT token with custom claim support.

```
{
  "iss": "<iss>",
  "prn": "<prn>",
  "iat": <iat>,
  "exp": <exp>",
  "nce": "<nce>",
  
  "first_name" : "Firstname",
  "last_name" : "Lastname",
  "display_name" : "displayname", 
  "avatar_url" : "https://example.com/image.jpg"
}
```

## User listing in Cloud Function

For app's requirements, app could use Cloud Function to support its own user listing functionality if it creates its own user table and implement some hooks.

1. Developer creates `user` table hosted in Cloud Function DB.
2. Developer supports `after_signup_sync` hooks.
3. Insert user attributes into `user` table in the hook.
4. Cloud Function could create its own query functionality with it's DB.

## [TBD] Use case: User Management Dashboard

Consider a dashboard, which provides general purpose user management functions. For normal application users, it should support:

- change avatar
- delete connected sessions
- update additional user information
- change password
- ...
 
For admin user, it should support:

- create user
- query users
- disable user
- send notification by user segment
- update additional user information
- reset user password
- ...

To support above functionalities, it would be provided by auth gear internally. It can connect auth gear DB directly, and no need to provide query interface to the public.

## [TBD] Auth UIKit

![](https://i.imgur.com/c4Vqk6G.png)

Consider skygear has a general purpose UIKit for user login/signup, the UI should response by following settings:

- auth criteria: username, email, phone number, ..., etc.
- auth protocols: SSO, LDAP, SAML, ..., etc.
- multi-factor authentication
- ...

![](https://i.imgur.com/0Rg7IOA.png)

It should be a separate service to handle such settings, and then Auth UIKit knows how to update its UI accordingly.
