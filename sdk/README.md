# rixl-csharp

C# client for the [RIXL](https://rixl.com) API.

[![NuGet](https://img.shields.io/nuget/v/Rixl.Sdk.svg)](https://www.nuget.org/packages/Rixl.Sdk)

## Install

```bash
dotnet add package Rixl.Sdk
```

Targets `.NET 8`. Brings in `Microsoft.Kiota.Bundle` (HTTP transport, serializers).

## Quick start

```csharp
using Rixl.Sdk;
using Microsoft.Kiota.Abstractions.Authentication;
using Microsoft.Kiota.Bundle;

var auth = new ApiKeyAuthenticationProvider(
    "YOUR_RIXL_API_KEY", "X-API-Key", ApiKeyLocation.Header);
var adapter = new DefaultRequestAdapter(auth);
var client = new RixlClient(adapter);

var image = await client.Images["PS5IMKoFLm"].GetAsync();
Console.WriteLine($"{image?.Id} {image?.Width}x{image?.Height}");
```

Default base URL: `https://api.rixl.com`. Override with `adapter.BaseUrl = "..."`.

## Resources

```csharp
// Feeds
var posts = await client.Feeds["FD4y3QB38S"].GetAsync();

// Images — list, get, delete
var page  = await client.Images.GetAsync();
var image = await client.Images["PS5IMKoFLm"].GetAsync();
await client.Images["PS5IMKoFLm"].DeleteAsync();

// Videos
var videos = await client.Videos.GetAsync();
var video  = await client.Videos["VI9VXQxWXQ"].GetAsync();
```

Upload (init → PUT bytes to presigned URL → complete) follows the same pattern as the other RIXL SDKs.

## Errors

API errors (400/401/403/404/500) are thrown as `Rixl.Sdk.Models.Github_com_rixlhq_api_internal_errors.ErrorResponse` exceptions; catch and inspect `Code` / `Error`.

## Support

Open an issue at [github.com/rixlhq/rixl-csharp](https://github.com/rixlhq/rixl-csharp/issues).
