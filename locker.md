# Locker
Locker simplifies authentication against Erste Group member servers. It allows developer to obtain access token for the user and store it in a secure manner. It also provides sensitive data secure storage and customizable user interface for authentication flow.

## Features
- [x] **Enrollment (Registration)** - Obtaining access token from oAuth2 serves for the user and enrolling the application to Locker server. 
- [x] **Strong Customer Authentication (Login)** - Obtaining access token after registration for the user using oAuth2 serves.
- [x] **Easy Access (Login)** - Obtaining access token after registration in exchange for the password.
- [x] **Lock (Logout)** - Securing the access token when the app is not in use.
- [x] **Revoke (Unregistration)** - Purging the access token from the device and revoking application from Locker server.
- [x] **Enable/Disable Easy Access** - Enabling/Disabling the password that secures the access token (token is available using oAuth2 server aka Strong Authentication after disabling Easy Access).
- [x] **Offline Verification** - Verifying user via Easy Access using offline mechanisms when application is in offline mode.
- [x] **Background Access** - Obtaining the scoped access token from background without user intervention.
- [x] **Encrypt/Decrypt Sensitive Data** - Securing sensitive data for various security levels. 

## Usual use case
The usual order of business with the LockerSDK is as follows:

1. Import - Import the LockerSDK and initialize the `ErsteGroupAuth` instance
2. Initialize - Configure the library by passing configuration dictionary
3. Check session status - Check auth session status. If the session status is not `.active`, call the `startAuthSession(...)` method to start a new `AuthSession`
4. Start the auth flow - Initiate the authentication UI if needed and obtain auth session with valid `AccessToken`.
5. Obtain token (with limited scope) without user interaction - Sometimes it is necessary to obtain read-only access token for dashboard widgets or similiar functionalities without active user interaction by calling `backgroundAccess(...)`.
6. End auth session - When the user is done with using the app. You can cancel his auth session by calling `endAuthSession(...)`.

## Usage reference

### Configuring locker
You need to configure the locker before you can use it.

```swift
    let locker = ErsteGroupAuth.shared.initializeLocker(...)
```

**See [configuration guide](configuration.md)** for all the available configuration options.

### Obtaining the current status of auth session
You can obtain the current session state at any time after the library is initialized either by calling `getAuthSessionStatus()`.

#### AuthSessionStatus

Auth session status is an object that describes current session status. This object is returned in the callback or as an return value in most of the methods of this framework.

Session status can be in one of these states:

* `Unregistered` - The user has not registered yet or the session has ended. By calling `startAuthSession(...)` user has to go through full authentication to create a new active session.
* `Enrolled` - The user is in enrolled state, accessToken is not available. By calling `startAuthSession(...)` user has to go through easy access to make the session active again
* `ActiveViaStrongAuth` - Session is active, accessToken is available. User was authenticated via strong customer authentication.
* `ActiveViaEasyAccess` - Session is active, accessToken is available. User was authenticated via easy access.
* `BackgroundAccess` - The descoped accessToken obtained without user interaction is available. By calling `startAuthSession(...)` user has to go through easy access to make the session active again and to obtain fully scoped token.
* `VerifiedOffline` - No access token is available. The user has been verified offline because the internet was not available. By calling `startAuthSession(...)` user has to go through easy access to make the session active again.

See [AuthSessionStatus.swift](../ErsteLockerSDK/AuthSessionStatus.swift) for further informations about status.

### Obtaining the access token
You can get the access token from any active `AuthSession` accessing its `accessToken` property.

There is no guarantee that the access token will not be expired by the time you access it. If that is the case, just call `startAuthSession(...)/backgroundAccess(...)` again to obtain a new one.

See [AccessToken.swift](../ErsteLockerSDK/AccessToken.swift) for further informations about access token.

### Starting auth flow
*Note: This method shows an UI to the User*

**You should call this function from the main thread**

You can use `startAuthSession(...)` method to start the authentication flow.

This attempts to obtain valid auth session via auth UI. Callback with the `AuthSessionStatus` will be called on main thread after the flow finishes.

It is ok to call this method event if the session is active if you need to get a new access token.

You should never call this method if `isUserInteractionInProgress()` is true. An error will be returned if you do.

```swift
// method signature
func startAuthSession(
    options : StartAuthSessionOptions?, 
    metadata : [String:Any?]?, 
    callback: @escaping (_ session:AuthSessionStatus, _ error : ErsteGroupAuthError?)->()
)
```

#### Parameters
* `options` - Additional configuration options for the flow like ui options, required authentication level, using cached secret flag (if it is enabled) and custom auth channel callback, which provides custom oAuth2 `code` and `codeVerifier`.
* `metadata` - Custom metadata that should be passed to server
* `callback` - Callback that will be called when the user goes through flow or when error occures. It contains `AuthSessionStatus` for the given scope and possible error that prevented user from going through auth flow. There are no guarantees that the result of this call will be active auth session. You can still get unregistered session status even that there was not error.

### Obtaining access token without user interaction
You can use `backgroundAccess(...)` to obtain access token without user interaction, even if the session is in enrolled state.

The given auth session must not be in unregistered state in order for this call to be successful.

Callback with the `AuthSessionStatus` will be called on main thread after the flow finishes.

```swift
// method signature
func backgroundAccess(
    metadata : [String:Any?]?, 
    callback: @escaping (_ session:AuthSessionStatus, _ error : BackgroundAccessError?) -> Void
)
```

#### Parameters:

* `metadata` - Custom metadata that should be passed to server
* `callback` - Callback that will be called when the background access obtains descoped token or ends with error. It contains current `AuthSessionStatus`. There are no guarantees that the result of this call will be background access auth session. You can get unregistered or enrolled session back.

### Ending auth session
Cancels the auth session.

You should call this function from the main thread and only if the user is not in unregistered auth session state. Otherwise callback will be returned with error.

```swift
// method signature
func endAuthSession(
    mode : EndAuthMode, 
    callback: @escaping (_ session:AuthSessionStatus, _ error : ErsteGroupAuthError?) -> ()
)
```

#### Parameters:

* `mode` - End auth session mode, which can be either `lock` or `destroy` in order to required session end.
* `callback` - Callback that will be called when the end auth session finishes. It contains current `AuthSessionStatus`.

### Dismissing the UI flow
There will be moments when the UI flow is required to cancel prematurely. You can dismiss any UI flow by calling `dismissUI(...)`.

Callback will be called when the UI is fully dismissed (or an error occured) and the interaction control is returned to your app.

You should call this function from the main thread and only if there is an active UI flow going on.

**Be aware that by calling this method, there is no guarantee that the `startAuthSession(...)` call that you started will not return in their callback a different `AuthSession` if they are in the middle of server interaction that cannot be cancelled.**

```swift
// method signature
func dismissUI(callback: @escaping (_ session:AuthSessionStatus, _ error : Error?)->())
```

### Revoking enrolled password
You can revoke your easy access password using `disableEasyAccess(...)`. User does not loose his enrollment, stays in enrolled state. This use case is usually used in application settings. It is necessary to go through strong customer authentication in order to get to active state after easy access was disabled.

Callback is called on main thread after password was revoked on server side.

This method can be called only in active state. 

```swift
// method signature
func disableEasyAccess(callback: @escaping (_ session:AuthSessionStatus, _ error : ErsteGroupAuthError?) -> Void)
```

### Enrolling new password
It is also possible to enroll new password after the old one was revoked without user reregistration by using `enableEasyAccess(...)` method. User is taken through strong customer authentication and password enrollment flow. 

Callback is returned after password enrollment flow is finished or canceled.

This method can be called only in active state. 

```swift
// method signature
func enableEasyAccess(callback: @escaping (_ session:AuthSessionStatus, _ error : ErsteGroupAuthError?) -> Void)
```

### Change easy access
Convenience method that combines revoking the enrolled easy access password and new enrollment.

```swift
// method signature
 func changeEasyAccess(callback: @escaping (_ session:AuthSessionStatus, _ error : ErsteGroupAuthError?) -> Void)
```


### Encryption of sensitive data
At some point, it is handy to encrypt user sensitive data before storing them. For this use case, `encrypt(...)` method was created. 

There are various levels of protection which depends on the session state application is in. For example secret data stored with `active` protection level are available only if app is in active state, obviously.

This method is synchronous.
 
```swift
// method signature
func encrypt(
    secret : String, 
    protectionLevel : ProtectionLevel
) throws -> String
```

#### Parameters:

* `secret` - Secret data string which is meant to be encrypted. 
* `protectionLevel` - Level of protection which describes in which auth session state is possible to decrypt encrypted sensitive data. There are `active`, `offline`, `background` and `base` protection levels which corresponds to appropriate auth session states.

### Decryption of sensitive data
In order to decrypt secrets encrypted by locker, `decrypt(...)` method should be used. 

App has to be in appropriate auth session state, for which sensitive data were encrypted (by selecting protection level during encryption).

```swift
// method signature
func decrypt(encryptedSecret : String) throws -> String
```

### Log messages handling
Application can implement `Logger` protocol in its `ApplicationDelegate`. If it does so, protocol's 
```swift
func handleLogMessage(_ message: String, level: LogLevel)
``` 
is called for each log event. It is then the app responsibility to record (or discard) log messages (perhaps based on combination of each message `level` and app's build state).

If this protocol is **not implemented** by the app `ApplicationDelegate`, then all log messages are printed using `NSLog()` as long as the app is build for `DEBUG`. If the `DEBUG` macro is not defined, all log messages are discarded. 

## Further reading

Please see [public API of the locker](../ErsteLockerSDK/ErsteLockerSDKAPI.swift) for interface documentation.
