# Rixl .NET SDK

[![NuGet](https://img.shields.io/nuget/v/Rixl.Sdk.svg)](https://www.nuget.org/packages/Rixl.Sdk)

The official .NET client for the [Rixl](https://rixl.com) API.

Rixl handles the media side of your product: uploading and delivering images
and videos, organising them into feeds and posts, and reporting on how people
engage with them. It also covers the account layer around that: users and
organisations, sign-in, subscriptions and invoices. This SDK gives you all of it
from C#, as a fluent request builder per path and a class for every request and
response body.

Targets .NET 8.

## Installation

```bash
dotnet add package Rixl.Sdk
```

That pulls in `Microsoft.Kiota.Bundle`, the runtime the generated code is built
on: HTTP transport plus the JSON, form, text and multipart serializers.

## Getting started

You build a client out of two pieces: something that authenticates requests, and
a request adapter that sends them. Then you point the adapter at the API:

```csharp
using Microsoft.Kiota.Abstractions.Authentication;
using Microsoft.Kiota.Bundle;
using Rixl.Sdk;

var auth = new ApiKeyAuthenticationProvider(
    Environment.GetEnvironmentVariable("RIXL_API_KEY")!,
    "X-API-Key",
    ApiKeyAuthenticationProvider.KeyLocation.Header);

var adapter = new DefaultRequestAdapter(auth) { BaseUrl = "https://api.rixl.com" };
var client = new RixlClient(adapter);

var projectId = Environment.GetEnvironmentVariable("RIXL_PROJECT_ID")!;
var page = await client.Media.V1.Projects[projectId].Images.GetAsync();

foreach (var image in page!.Images!)
{
    Console.WriteLine(image.Id);
}
```

The adapter has no base URL of its own, so setting `BaseUrl` is not optional.
do it before you make a request, and point it somewhere else when you are
testing against another environment.

Every call is async, takes an optional `CancellationToken`, and returns a
nullable model.

## Authentication

There are two ways to identify yourself, and they answer different questions.

### API keys, for your backend calling as itself

An API key represents your organisation. Use it for work your own systems do:
importing a catalogue, running a nightly report, reconciling invoices. Keep it
out of source control and read it from configuration or the environment:

```csharp
var auth = new ApiKeyAuthenticationProvider(
    apiKey, "X-API-Key", ApiKeyAuthenticationProvider.KeyLocation.Header);
```

The key travels as the `X-API-Key` header. Anyone holding it can do anything
your organisation can, so it belongs on a server. Never put one in a browser, a mobile
app, or anything you ship to users.

### Client credentials, for acting on behalf of your users

If you are building on top of Rixl and your own users each need their own slice
of it, use client credentials. You exchange a client ID and secret for a
short-lived token scoped to a single end user, so one customer can never read
another's media.

Create the credential with an API-key client. The secret comes back once:

```csharp
using Rixl.Sdk.Models.Clientauth.V1;

var created = await client.Platform.Clientauth.V1.Credentials.PostAsync(
    new CreateClientCredentialRequest
    {
        Name = "Production backend",
        OrgId = orgId,
    });

Console.WriteLine(created!.Credential!.ClientId);
Console.WriteLine(created.ClientSecret);
```

Then mint a token per user. `Subject` is your own identifier for that person,
whatever your database calls them:

```csharp
var token = await client.Platform.Clientauth.V1.Token.PostAsync(
    new MintClientTokenRequest
    {
        ClientId = clientId,
        ClientSecret = clientSecret,
        Subject = user.Id,
        ProjectId = projectId,
        TtlMinutes = 15,
    });
```

Tokens last at most 15 minutes and there is no refresh token. When one expires
you mint another. Nothing in the SDK does that for you, so wrap the mint call in
an `IAccessTokenProvider` and let the bearer provider ask for a token whenever
it needs one:

```csharp
var bearer = new BaseBearerTokenAuthenticationProvider(tokenProvider);
var userAdapter = new DefaultRequestAdapter(bearer) { BaseUrl = "https://api.rixl.com" };
var userClient = new RixlClient(userAdapter);
```

`IAccessTokenProvider` wants two members. `GetAuthorizationTokenAsync` is where
you return `AccessToken` from a mint call and cache it until `ExpiresAt`, and
`AllowedHostsValidator` can return `new AllowedHostsValidator()` to allow every
host. Tokens go out as `Authorization: Bearer`.

Credentials are managed through the same builder you created them with:
`Credentials.GetAsync()` lists them and
`Credentials[credentialId].Revoke.PostAsync()` kills one. Revoking stops new
tokens immediately; ones already issued die within 15 minutes.

### Public endpoints

Some reads need no credentials at all: fetching a public image or video,
reading a public feed, listing supported languages, and the sign-in flows under
`/auth/v1`. Point an anonymous provider at those:

```csharp
var adapter = new DefaultRequestAdapter(new AnonymousAuthenticationProvider())
{
    BaseUrl = "https://api.rixl.com",
};
var client = new RixlClient(adapter);

var image = await client.Media.V1.Images[imageId].GetAsync();
var languages = await client.Media.V1.Languages.GetAsync();
var feed = await client.Posts.V1.Feeds[feedId].GetAsync();
```

Mind the difference between the two image paths: `Media.V1.Images` is the public
read, while `Media.V1.Projects[projectId].Images` is the authenticated
collection you list, upload to and delete from.

## What you can do

Every area of the API is a property on the client, and the path you type mirrors
the URL.

**Media**: `client.Media.V1`. `Images` and `Videos` for public reads, and
`Projects[projectId]` for everything else: listing, uploading, deleting,
visibility, plus `AudioTracks`, `Chapters` and `Subtitles` on a video.
`Languages` lists what you can localise into.

**Content**: `client.Posts.V1` for posts and feeds,
`client.Feeds.V1.Projects[projectId].Feeds` for feed configuration, and
`client.Organizations[orgId].Projects` for the projects everything else hangs
off. That is why so many calls take a project ID.

**Analytics**: `client.Analytics.V1`: `Dashboard`, `Events`, `Posts`, `Videos`,
`Feeds`, `Funnels`, `Retention`, `Realtime`, `Top`. Track events and read back
engagement, playback and live activity.

**Billing**: `client.Billing.V1`: `Plans`, `Subscription`, `Invoices`,
`PaymentMethods`, `Checkout`, `StorageUsage`, `BandwidthUsage`, `Tax`,
`Address`.

**Account management**: `client.Auth.V1`: `Register`, `Login`, `Token`, `Users`,
`Passkey`, `Password`, `Providers`, `Memberships`, `Policies`, `Email`, `Blog`.
Sign-in flows including passkeys and TOTP, organisation membership and roles,
and transactional email.

**Platform**: `client.Platform` for `Auth.V1` and `Clientauth.V1`, and
`client.Organizations[orgId].ApiKeys` for API keys.

`client.Internal` is storage-callback plumbing that Rixl calls itself. You
should not need it.

## Working with resources

Builders compose the same way everywhere, so once you have used one you have
used all of them. Reads and deletes:

```csharp
var images = client.Media.V1.Projects[projectId].Images;

var page = await images.GetAsync();
await images[imageId].DeleteAsync();
```

Calls that send data take a generated body class:

```csharp
using Rixl.Sdk.Media.V1.Projects.Item.Images.Upload;

var upload = await images.Upload.PostAsync(new UploadPostRequestBody
{
    Name = "photo.jpg",
    ProjectId = projectId,
});
```

Everything is nullable. A property you never set is left out of the request
rather than sent empty, and a property the API omits comes back `null`, so check
before you dereference.

## Uploading files

Uploads happen in two steps. You ask Rixl for a URL, then send the bytes
straight to storage. The bytes never pass through the API, so large files stay fast:

```csharp
var upload = await images.Upload.PostAsync(body);

using var http = new HttpClient();
using var content = new StreamContent(File.OpenRead("photo.jpg"));
content.Headers.ContentType = new MediaTypeHeaderValue("image/jpeg");

var response = await http.PutAsync(upload!.UploadUrl, content);
response.EnsureSuccessStatusCode();
```

Videos work the same way through `Videos.Upload`, except the response gives you
two URLs: `VideoUploadUrl` for the file and `PosterUploadUrl` for its poster
image.

There is no "finish" call to make. Storage tells Rixl when the object lands and
the image or video becomes available on its own.

## Pagination

List calls take a limit and an offset, set through the request configuration:

```csharp
var limit = 50;
var offset = 0;

while (true)
{
    var page = await images.GetAsync(config =>
    {
        config.QueryParameters.PaginationLimit = limit;
        config.QueryParameters.PaginationOffset = offset;
    });

    if (page?.Images is null || page.Images.Count == 0)
    {
        break;
    }

    foreach (var image in page.Images)
    {
        Console.WriteLine(image.Id);
    }

    offset += limit;
}
```

Nothing pages for you. Ask for the next offset yourself. Responses also carry
`Total`, but it deserialises as an `UntypedNode` rather than a number, so
stopping on a short page is the simpler test.

## Handling errors

Anything that is not a 2xx is thrown as
`Microsoft.Kiota.Abstractions.ApiException`, which carries the status code and
the response headers:

```csharp
using Microsoft.Kiota.Abstractions;

try
{
    var image = await client.Media.V1.Images[imageId].GetAsync();
}
catch (ApiException e)
{
    Console.Error.WriteLine($"rixl returned {e.ResponseStatusCode}");
    throw;
}
```

What the codes mean:

| Status | What happened | What to do |
| --- | --- | --- |
| 400 | The request was malformed or failed validation | Fix the request; retrying will not help |
| 401 | The key or token is missing, expired or invalid | Check the credential |
| 403 | The credential is valid but not allowed to do this | Check what it is permitted to do |
| 404 | No such resource, or it belongs to another organisation | Check the ID and the project |
| 429 | You are going too fast | Back off and retry |
| 5xx | Something broke on our side | Retry with backoff |

Connection failures and timeouts surface as `HttpRequestException` or
`TaskCanceledException`, not as `ApiException`.

## Timeouts

Requests go out through whatever `HttpClient` you hand the adapter, so timeouts
stay yours to control:

```csharp
var http = new HttpClient { Timeout = TimeSpan.FromSeconds(30) };
var adapter = new DefaultRequestAdapter(auth, null, null, http)
{
    BaseUrl = "https://api.rixl.com",
};
```

Per call, pass a `CancellationToken` instead. Every method takes one as its
last argument. A `DelegatingHandler` on that `HttpClient` is where tracing
headers go if you want them on every outbound request.

## Versioning

This package follows [SemVer](https://semver.org/spec/v2.0.0.html). New API
resources arrive in minor releases; renamed or removed operations only in major
ones. If an upgrade breaks you unexpectedly, please open an issue. We would
rather hear about it.

## Support

Bugs and feature requests:
[github.com/rixlhq/rixl-csharp/issues](https://github.com/rixlhq/rixl-csharp/issues).
