## Using Amazon Cognito Hosted UI

Amazon Cognito User Pools provides a customizable user experience via a web "Hosted UI". The Hosted UI provides an OAuth 2.0 flow that allows you to launch a web view (without embedding an SDK for Cognito or a social provider) via your application. The Hosted UI allows end-users to login and register directly to your user pool, through Facebook, Amazon, and Google, as well as through OpenID Connect (OIDC) and SAML identity providers. The Amplify CLI will setup and configure a Hosted UI for you when adding Authentication to your app. [Learn More](~/cli/auth/overview.md)

### Automated Setup with CLI

When configuring Authentication with the CLI, you can choose an identity provider (Google, Facebook or Login with Amazon) or use Cognito User Pools directly. When setting up the social providers, you will need the domain name that is generated by Cognito User Pools, as well as the credentials retrieved from the social providers website. 

Run the following command in your project’s root folder:

```bash
amplify add auth
```

Select Default configuration with Social Provider (Federation):

```console
Do you want to use the default authentication and security configuration? 
  Default configuration 
❯ Default configuration with Social Provider (Federation) 
  Manual configuration 
  I want to learn more.
```

You will be asked for the credentials from your social provider setup:
 - [Set up auth with Facebook](#set-up-auth-with-facebook)
 - [Set up auth with Google](#set-up-auth-with-google)
 - [Set up auth with Amazon](#set-up-auth-with-amazon)
 - [Set up auth with Apple](#set-up-auth-with-apple)

After going through the flow, run the following command to deploy the configured resources to the cloud:

```bash
amplify push
```
You will find a domain-name provisioned by the CLI for the hosted UI as an output in the terminal. You can also find that information anytime later using the `amplify status` command.

> Your Cognito User Pool domain is structured like so: `<unique_domain_prefix>-<env-name>.auth.<region>.amazoncognito.com`. Use this domain in the above steps when setting up your social provider.

### Set up auth with Facebook

Set up your app in Facebook by following Facebook's [App Development](https://developers.facebook.com/docs/apps) guide. Facebook will provide you an **App ID** and **App Secret** for your application which you will find in the app's **Basic Settings**; Take note of these.

When configuring your app, use the following settings (the domain will be generated in the next section):

- When setting up scenarios, choose **Facebook Login**
- Add your Cognito User Pools domain with the */oauth2/idpresponse*:
    - `https://<your-user-pool-domain>/oauth2/idpresponse` to:
        - **Facebook Login > Settings > Valid OAuth Redirect URIs**
        - **Platforms > Website > Site URL**

### Set up auth with Google

Set up and create a *Web application* in Google by following Google's [Setting up OAuth 2.0](https://support.google.com/googleapi/answer/6158849) doc. Google will provide you a a **Client ID** and **Client Secret** for your application; take note of these.

- On the **OAuth consent screen**, under Authorized domains, add `amazoncognito.com`
- When setting up the **Web application** add the following:
    - **Name Field**: The name of your application
    - **Authorized JavaScript origins**: `amazoncognito.com`
    - **Authorized redirect URIs**: `https://<your-user-pool-domain>/oauth2/idpresponse`

### Set up auth with Amazon

- Sign in to the Develop Portal at [Login with Amazon](http://login.amazon.com/), select **App Console** using your Amazon.com credentials.
- Choose **Create a Security Profile** or choose an existing one that you have created.
    - **Security Profile Name**: Your application name
    - **Security Profile Description**: Your application description
    - **Consent Privacy Notice URL**: A URL to your privacy notice
- Click **Show Client ID and Client Secret** and take note of these values.

### Set up auth with Apple

> The [Apple App Store Developer Guidelines](https://developer.apple.com/app-store/review/guidelines/#sign-in-with-apple) require that Sign in with Apple to be available in applications that ***exclusively*** use third-party sign-in options, such as Facebook or Google. If you are only or additionally using Cognito User Pools directly (Signing in with Email/Password), you are not required to include Sign in with Apple.

Before you add support for Signin with Apple you will need an Apple Developer account, which is a paid account with Apple.

Learn more about [How to set up Sign in with Apple for Amazon Cognito](https://aws.amazon.com/blogs/security/how-to-set-up-sign-in-with-apple-for-amazon-cognito/). You can then use `AWSMobileClient` to launch the Hosted UI directly to sign in with Apple by specifying `HostedUIOptions(identityProvider: "SignInWithApple")`

### Manual Setup

To configure your application for hosted UI, you need to use *HostedUI* options. Update your `awsconfiguration.json` file to add a new configuration for `Auth`. The configuration should look like this:

    ```json
    {
        "IdentityManager": {
            ...
        },
        "CredentialsProvider": {
            ...
        },
        "CognitoUserPool": {
            ...
        },
        "Auth": {
            "Default": {
                "OAuth": {
                    "WebDomain": "YOUR_AUTH_DOMAIN.auth.us-west-2.amazoncognito.com", // Do not include the https:// prefix
                    "AppClientId": "YOUR_APP_CLIENT_ID",
                    "AppClientSecret": "YOUR_APP_CLIENT_SECRET",
                    "SignInRedirectURI": "myapp://",
                    "SignOutRedirectURI": "myapp://",
                    "Scopes": ["openid", "email"]
                }
            }
        }
    }
    ```

> Note: The User Pool OIDC JWT token obtained from a successful sign-in will be federated into a configured Cognito Identity Pool. The SDK will automatically exchange this information with Cognito in order to retrieve temporary AWS credentials (Access Key ID and Secret Key).

### Setup Cognito Hosted UI in your iOS App

1. Add `myapp://` to your app's URL schemes:

    Right-click Info.plist and then choose Open As > Source Code.

    Add the following entry in URL scheme:

    ```xml
        <plist version="1.0">

        <dict>
        <!-- YOUR OTHER PLIST ENTRIES HERE -->

        <!-- ADD AN ENTRY TO CFBundleURLTypes for Cognito Auth -->
        <!-- IF YOU DO NOT HAVE CFBundleURLTypes, YOU CAN COPY THE WHOLE BLOCK BELOW -->
        <key>CFBundleURLTypes</key>
        <array>
            <dict>
                <key>CFBundleURLSchemes</key>
                <array>
                    <string>myapp</string>
                </array>
            </dict>
        </array>

        <!-- ... -->
        </dict>
    ```

### Launching the Hosted UI

To launch the Hosted UI from from your application, you can use the `showSignIn` API of `AWSMobileClient.default()`:

```swift
// Optionally override the scopes based on the usecase.
let hostedUIOptions = HostedUIOptions(scopes: ["openid", "email"])

// Present the Hosted UI sign in.
AWSMobileClient.default().showSignIn(navigationController: self.navigationController!, hostedUIOptions: hostedUIOptions) { (userState, error) in
    if let error = error as? AWSMobileClientError {
        print(error.localizedDescription)
    }
    if let userState = userState {
        print("Status: \(userState.rawValue)")
    }
}
```

> Optional: If your app deployment target is < iOS 11, add the following callback in your App Delegate's `application:open url` method. This callback is required, because `SFSafariViewController` is used in < iOS 11 to integrate with the Hosted UI.

```swift
func application(_ application: UIApplication, open url: URL, sourceApplication: String?, annotation: Any) -> Bool {
    AWSMobileClient.default().handleAuthResponse(application, open: url, sourceApplication: sourceApplication, annotation: annotation)
    return true
}
```

**Note:** By default, the Hosted UI will show all login options; the username-password flow as well as any social providers which are configured. If you wish to bypass the extra sign-in screen showing all the provider options and launch your desired social provider login directly, you can set the `HostedUIOptions` as shown in the next section.

## Directly Launching Facebook, Google, or SAML

You can by-pass the Cognito Hosted UI screen and navigate directly to and from a third provider authentication page. This allows you to login with a social provider configured in Cognito User Pools while not displaying Cognito's Hosted UI (e.g. showing a *Login with Facebook* button in your application).

```swift
// Option to launch Google sign in directly
let hostedUIOptions = HostedUIOptions(scopes: ["openid", "email"], identityProvider: "Google")
//  OR
// Option to launch Facebook sign in directly
let hostedUIOptions = HostedUIOptions(scopes: ["openid", "email"], identityProvider: "Facebook")
//  OR
// Option to launch Apple sign in directly
let hostedUIOptions = HostedUIOptions(identityProvider: "SignInWithApple")

// Present the Hosted UI sign in.
AWSMobileClient.default().showSignIn(navigationController: self.navigationController!, hostedUIOptions: hostedUIOptions) { (userState, error) in
    if let error = error as? AWSMobileClientError {
        print(error.localizedDescription)
    }
    if let userState = userState {
        print("Status: \(userState.rawValue)")
    }
}
```

### Signing out via the Hosted UI

```swift
// Setting invalidateTokens: true will make sure the tokens are invalidated
AWSMobileClient.default().signOut(options: SignOutOptions(invalidateTokens: true)) { (error) in
    print("Error: \(error.debugDescription)")
}
```

If you want to sign out locally by just deleting tokens, you can call `signOut` method:

```swift
AWSMobileClient.default().signOut()
```

## Set up Auth with Auth0 

You can use `AWSMobileClient` to use `Auth0` as an `OAuth 2.0` provider. 
You use `Auth0` as an identity provider for a Cognito Federated Identity Pool. This will allow users authenticated via Auth0 to have access to your AWS resources. Learn [how to integrate Auth0 with Cognito Federated Identity Pools](https://auth0.com/docs/integrations/integrating-auth0-amazon-cognito-mobile-apps)

### Configure your iOS App

1. To configure your application for hosted UI, you need to use *HostedUI* options. Update your `awsconfiguration.json` file to add a new configuration for `Auth`. The configuration should look like this:

    ```json
    {
        "IdentityManager": {
            ...
        },
        "CredentialsProvider": {
            ...
        },
        "CognitoUserPool": {
            ...
        },
        "Auth": {
            "Default": {
                "OAuth": {
                    "AppClientId": "YOUR_AUTH0_APP_CLIENT_ID",
                    "WebDomain": "YOUR_AUTH0_DOMAIN.auth0.com", // Do not include the https:// prefix
                    "TokenURI": "https://YOUR_AUTH0_DOMAIN.auth0.com/oauth/token",
                    "SignInURI": "https://YOUR_AUTH0_DOMAIN.auth0.com/authorize",
                    "SignInRedirectURI": "com.your.bundle.configured.in.auth0://YOUR_AUTH0_DOMAIN.auth0.com/ios/com.your.bundle/callback",
                    "SignOutURI": "https://YOUR_AUTH0_DOMAIN.auth0.com/v2/logout",
                    "SignOutURIQueryParameters": {
                        "client_id" : "YOUR_AUTH0_APP_CLIENT_ID",
                        "returnTo" : "com.your.bundle.configured.in.auth0://yourserver.auth0.com/ios/com.amazonaws.AWSAuthSDKTestApp/callback"
                    },
                    "Scopes": ["openid", "email"]
                }
            }
        }
    }
    ```

1. Add the signin and signout redirect URIs to your app's URL schemes:

    Right-click Info.plist and then choose Open As > Source Code.

    Add the following entry in URL scheme:

    ```xml
        <plist version="1.0">

        <dict>
        <!-- YOUR OTHER PLIST ENTRIES HERE -->

        <!-- ADD AN ENTRY TO CFBundleURLTypes for Auth0 -->
        <!-- IF YOU DO NOT HAVE CFBundleURLTypes, YOU CAN COPY THE WHOLE BLOCK BELOW -->
        <key>CFBundleURLTypes</key>
        <array>
            <dict>
                <key>CFBundleURLSchemes</key>
                <array>
                    <string>com.your.bundle.configured.in.auth0://yourserver.auth0.com/ios/com.amazonaws.AWSAuthSDKTestApp/callback</string>
                </array>
            </dict>
        </array>

        <!-- ... -->
        </dict>
    ```

### Launch the Hosted UI for Auth0

To launch the Hosted UI from from your application, you can use the `showSignIn` API of `AWSMobileClient.default()`:

```swift
// Specify the scopes and federation provider name.
 let hostedUIOptions = HostedUIOptions(scopes: ["openid", "email"], federationProviderName: "YOUR_AUTH0_DOMAIN.auth0.com")

// Present the Hosted UI sign in.
AWSMobileClient.default().showSignIn(navigationController: self.navigationController!, hostedUIOptions: hostedUIOptions) { (userState, error) in
    if let error = error as? AWSMobileClientError {
        print(error.localizedDescription)
    }
    if let userState = userState {
        print("Status: \(userState.rawValue)")
    }
}

// Present the Hosted UI sign in.
AWSMobileClient.default().showSignIn(navigationController: self.navigationController!, hostedUIOptions: hostedUIOptions) { (userState, error) in
    if let error = error as? AWSMobileClientError {
        print(error.localizedDescription)
    }
    if let userState = userState {
        print("Status: \(userState.rawValue)")
    }
}
```

> Optional: If your app deployment target is < iOS 11, add the following callback in your App Delegate's `application:open url` method. This callback is required, because `SFSafariViewController` is used in < iOS 11 to integrate with the Hosted UI.

```swift
func application(_ application: UIApplication, open url: URL, sourceApplication: String?, annotation: Any) -> Bool {
    return AWSMobileClient.default().handleAuthResponse(application, open: url, sourceApplication: sourceApplication, annotation: annotation)
}
```

### Signing Out via the Hosted UI

```swift
// Setting invalidateTokens: true will make sure the tokens are invalidated
AWSMobileClient.default().signOut(options: SignOutOptions(invalidateTokens: true)) { (error) in
    print("Error: \(error.debugDescription)")
}
```

If you want to sign out locally by just deleting tokens, you can call `signOut` method:

```swift
AWSMobileClient.default().signOut()
```