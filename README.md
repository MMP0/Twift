# Twift

[![Twitter API v2 badge](https://img.shields.io/endpoint?url=https%3A%2F%2Ftwbadges.glitch.me%2Fbadges%2Fv2)](https://developer.twitter.com/en/docs/twitter-api/early-access)
[![Documentation Coverage](https://github.com/daneden/Twift/blob/badges/.github/badges/coverage.svg)](https://github.com/daneden/Twift/wiki)

Twift is an asynchronous Swift library for the Twitter v2 API.

- [x] No external dependencies
- [x] Only one callback-based method ([`Authentication.requestUserCredentials`](https://github.com/daneden/Twift/wiki/Twift_Authentication#requestusercredentialsclientcredentialscallbackurlpresentationcontextproviderwith))
- [x] Full Swift type definitions/wrappers around Twitter's API objects

## Quick Start Guide

New `Twift` instances must be initiated with either User Access Tokens or an App-Only Bearer Token:

```swift
// User access tokens
let clientCredentials = OAuthToken(key: CONSUMER_KEY, secret: CONSUMER_SECRET)
let userCredentials = OAuthToken(key: ACCESS_KEY, secret: ACCESS_SECRET)
let userAuthenticatedClient = Twift(.userAccessTokens(clientCredentials: clientCredentials, userCredentials: userCredentials)

// Bearer token
let appOnlyClient = Twift(.appOnly(bearerToken: BEARER_TOKEN)
```

You can acquire user access tokens by authenticating the user with `Twift.Authentication().requestUserCredentials()`:

```swift
var client: Twift?

Twift.Authentication().requestUserCredentials(
  clientCredentials: clientCredentials,
  callbackURL: URL(string: "twift-test://")!
) { (userCredentials, error) in
  if let creds = userCredentials {
    client = Twift(.userAccessTokens(clientCredentials: clientCredentials, userCredentials: creds))
  }
}
```

Once initiated, you can begin calling methods appropriate for the authentication type:

```swift
do {
  // User objects always return id, name, and username properties,
  // but additional properties can be requested by passing a `fields` parameter
  let authenticatedUser = try await userAuthenticatedClient.getMe(fields: [\.profilePhotoUrl, \.description])
  
  // Non-standard properties are optional and require unwrapping
  if let description = authenticatedUser.description {
    print(description)
  }
} catch {
  print(error.localizedDescription)
}
```

## Requirements

> To be completed

## Documentation

You can find the full documentation in [this repo's Wiki](https://github.com/daneden/Twift/wiki) (auto-generated by [SwiftDoc](https://github.com/SwiftDoc/swift-doc)). 

## Quick Tips

### Typical Method Return Types
Twift's methods generally return `TwitterAPI[...]` objects containing up to four properties:

- `data`, which contains the main object(s) you requested (e.g. for the `getUser` endpoint, this contains a `User`)
- `includes`, which includes any expansions you request (e.g. for the `getUser` endpoint, you can optionally request an expansion on `pinnedTweetId`; this would result in the `includes` property containing a `Tweet`)
- `meta`, which includes information about pagination (such as next/previous page tokens and result counts)
- `errors`, which includes an array of non-failing errors

All of the methods are throwing, and will throw either a `TwiftError`, indicating a problem related to the Twift library (such as incorrect parameters passed to a method) or a `TwitterAPIError`, indicating a problem sent from Twitter's API as a response to the request.

###  Fields and Expansions

Many of Twift's methods accept two optional parameters: `fields` and `expansions`. These parameters allow you to request additional `fields` (properties) on requested objects, as well as `expansions` on associated objects. For example:

```swift
// Returns the currently-authenticated user
let response = try? await userAuthenticatedClient.getMe(
  // Asks for additional fields: the profile image URL, and the user's description/bio
  fields: [\.profileImageUrl, \.description],
  
  // Asks for expansions on associated fields; in this case, the pinned Tweet ID.
  // This will result in a Tweet on the returned `TwitterAPIDataAndIncludes.includes`
  expansions: [
    // Asks for additional fields on the Tweet: the Tweet's timestamp, and public metrics (likes, retweets, and replies)
    .pinnedTweetId([
      \.createdAt,
      \.publicMetrics
    ])
  ]
)

// The user object
let me = response?.data

// The user's pinned Tweet
let tweet = response?.includes?.tweets.first
```

### Optional Actor IDs

Many of Twift's methods require a `User.ID` in order to make requests on behalf of that user. For convenience, this parameter is often marked as optional, since the currently-authenticated `User.ID` may be found on the instance's authentication type:

```swift
var client: Twift?
var credentials: OAuthToken?

Twift.Authentication().requestUserCredentials(
  clientCredentials: clientCredentials,
  callbackURL: URL(string: "twift-test://")!
) { (userCredentials, error) in
  if let userCredentials = userCredentials {
    client = Twift(.userAccessTokens(clientCredentials: clientCredentials, userCredentials: userCredentials))
    credentials = userCredentials
  }
}

// Elsewhere in the app...

// These two calls are equivalent since the client was initialized with an OAuthToken containing the authenticated user's ID
let result1 = try? await client?.followUser(sourceUserId: credentials.userId!, targetUserId: "12")
let result2 = try? await client?.followUser(targetUserId: "12")

```
