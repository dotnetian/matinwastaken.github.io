---
title: Implementing Limited Speed Download with Pause/Resume Capability in ASP.NET Core
date: 2023-08-23 13:15:00 +0330
author: matin
categories: [Web Development]
tags: [csharp, dotnet, aspnetcore]
---

## Introduction

The LimitedSpeedDownloadResult class is a custom implementation in ASP.NET Core, providing limited speed download functionality with pause/resume capability. This article aims to thoroughly explain the code, its development process, challenges faced, and more. 

## Code Overview

Let's start by examining the code in detail, step by step.

```csharp
using Microsoft.AspNetCore.Mvc;
using System.Collections.Concurrent;
using System.Web;
```

The code includes the necessary namespaces: `Microsoft.AspNetCore.Mvc` for working with ASP.NET Core and `System.Collections.Concurrent` for using concurrent dictionaries, as well as `System.Web` for encoding file names.

```csharp
public class LimitedSpeedDownloadResult : IActionResult
{
    private static readonly ConcurrentDictionary<string, int> _sessionCounts = new ConcurrentDictionary<string, int>();
    private readonly string _filePath;
    private readonly int _kBps;
    private readonly int _maxConcurrentSessions;

    public LimitedSpeedDownloadResult(string filePath, int kbps, bool bypassTempLink = false)
    {
        _filePath = filePath.Replace('\\', '/');
        _maxConcurrentSessions = bypassTempLink ? 2 : 1;
        _kBps = bypassTempLink ? kbps / 2 : kbps;
    }
```

The code declares a class named `LimitedSpeedDownloadResult`, which implements the `IActionResult` interface. This class represents the limited speed download result with pause/resume capability.

The class contains the following private fields:
- `_sessionCounts`: A static concurrent dictionary used to store the session counts for each client IP address.
- `_filePath`: A string that represents the file path of the download.
- `_kBps`: An integer indicating the download speed in kilobytes per second.
- `_maxConcurrentSessions`: An integer specifying the maximum number of concurrent sessions allowed per client IP address.

A constructor is defined to initialize the instance of the `LimitedSpeedDownloadResult` class. The constructor takes three parameters:
- `filePath`: The file path of the download.
- `kbps`: The download speed in kilobytes per second.
- `bypassTempLink` (optional): A flag indicating whether to bypass temporary link restrictions, as because we are limiting the maximum count of concurrent sessions to 1, some complex download managers might consider the download link as an one-time link, and disable pause/resume feature in their interface. Setting this to true, will set the `_maxConcurrentSessions` to 2, and dividing the download speed by 2, to balance the download speed and prevent doubling it by download managers, caused by sending more than one concurrent requests. NOTE that this will make the download speed on normal downloaders half of the value specified. It defaults to `false`.

Now, let's proceed to the next part of the code.

```csharp
public async Task ExecuteResultAsync(ActionContext context)
{
    FileStream stream = new FileStream(_filePath, FileMode.Open, FileAccess.Read, FileShare.ReadWrite);
    BinaryReader reader = new BinaryReader(stream);

    long fileLength = stream.Length;
    int packSize = _kBps * 128;

    string clientIp = context.HttpContext.Connection.RemoteIpAddress.ToString();

    if (_sessionCounts.ContainsKey(clientIp) && _sessionCounts[clientIp] >= _maxConcurrentSessions)
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        return;
    }

    _sessionCounts.AddOrUpdate(clientIp, 1, (key, value) => value + 1);

    string fileName = "SkyBlock.mp4";
    context.HttpContext.Response.Headers.Add("Content-Disposition", $"attachment; filename={fileName}");
    context.HttpContext.Response.Headers.Add("Accept-Ranges", "bytes");
```

In the `ExecuteResultAsync` method, the code starts by opening a file stream and creating a binary reader based on the specified file path. It also retrieves the file length and calculates the packet size based on the download speed.

Next, the client's IP address is obtained from the HttpContext to track the sessions. If the session count for the client's IP address exceeds the maximum allowed concurrent sessions, a `429 Too Many Requests` response is sent, indicating the client has initiated too many concurrent downloads.

If the session count is within the allowed limit, the session count is incremented for the client and the `Content-Disposition` and `Accept-Ranges` headers are set in the response.

Let's proceed to the next part of the code.

```csharp
try
{
    if (context.HttpContext.Request.Headers.ContainsKey("Range"))
    {
        string rangeHeader = context.HttpContext.Request.Headers["Range"].ToString();
        string[] rangeValues = rangeHeader.Replace("bytes=", "").Split("-");

        long startByte = Convert.ToInt64(rangeValues[0]);
        long endByte = fileLength - 1;

        if (rangeValues.Length > 1 && !string.IsNullOrEmpty(rangeValues[1]))
        {
            endByte = Convert.ToInt64(rangeValues[1]);
        }

        context.HttpContext.Response.StatusCode = 206;
        context.HttpContext.Response.Headers.Add("Content-Range", $"bytes {startByte}-{endByte}/{fileLength}");
        context.HttpContext.Response.Headers.Add("Content-Length", (endByte - startByte + 1).ToString());

        stream.Seek(startByte, SeekOrigin.Begin);
        packSize = (int)Math.Min(packSize, endByte - startByte + 1);
        int packsCount = (int)Math.Ceiling((double)(endByte - startByte + 1) / packSize);

        for (int i = 0; i < packsCount; i++)
        {
            byte[] buffer = reader.ReadBytes(packSize);
            await context.HttpContext.Response.Body.WriteAsync(buffer, 0, buffer.Length, context.HttpContext.RequestAborted);
            int delay = (int)Math.Ceiling(1000.0 * buffer.Length / (_kBps * 1024));
            await Task.Delay(delay, context.HttpContext.RequestAborted);
        }
    }
    else
    {
        context.HttpContext.Response.Headers.Add("Content-Length", fileLength.ToString());
        int packsCount = (int)Math.Ceiling((double)fileLength / packSize);

        for (int i = 0; i < packsCount; i++)
        {
            byte[] buffer = reader.ReadBytes(packSize);
            await context.HttpContext.Response.Body.WriteAsync(buffer, 0, buffer.Length, context.HttpContext.RequestAborted);
            int delay = (int)Math.Ceiling(1000.0 * buffer.Length / (_kBps * 1024));
            await Task.Delay(delay, context.HttpContext.RequestAborted);
        }
    }

    context.HttpContext.Response.ContentType = "application/octet-stream";
    string utf8EncodingFileName = HttpUtility.UrlEncode(_filePath.Split("/").Last(), System.Text.Encoding.UTF8);
    context.HttpContext.Response.Headers.Add("Content-Disposition", "attachment;filename=" + utf8EncodingFileName);
}
catch (OperationCanceledException)
{
    _sessionCounts.AddOrUpdate(clientIp, 0, (_, value) => Math.Max(0, value - 1));
}
finally
{
    reader.Close();
    stream.Close();
}
```

In the try block of the code, it checks if the request contains the `Range` header. If present, it parses the range values to determine the start and end bytes of the requested range. It then sets the appropriate response headers (`Content-Range` and `Content-Length`) to indicate the partial content being served.

The code seeks to the start byte position in the file stream and determines the number of packets required to serve the requested range. It proceeds to read and write each packet to the response stream, controlling the speed by introducing a delay between packets.

If the request doesn't contain the `Range` header, the code goes to the else block and serves the entire file without any specific range.

After that, the `Content-Type` header is set to `application/octet-stream`, indicating a binary file download. The file name is also URL-encoded using `HttpUtility.UrlEncode` to ensure compatibility with various client browsers and prevent potential encoding issues.

In case the request is canceled by the client, an `OperationCanceledException` is caught, and the session count for the client is decremented to reflect the aborted request and remove active session assigned for that IP address, makes the client able to resend the request later, which is necessary for pause/resume implementation.

When done, in the `finally` block, the code ensures that the reader and stream are closed to release the resources properly.

## Benefits and Pros

Implementing the LimitedSpeedDownloadResult class in ASP.NET Core offers several benefits and advantages for websites and organizations:

1. **Controlled Server Load:** By limiting the download speed of files, the server can effectively manage its resources and mitigate the risk of overload. This approach ensures that the server bandwidth is distributed evenly among all users, preventing a single user from consuming excessive resources and negatively impacting the performance of the website.

2. **Enhanced User Experience:** Limited speed downloads with pause/resume capability provide a better user experience. Users can initiate downloads, pause them if needed, and resume from where they left off without starting the download from the beginning. This feature is particularly useful for large file downloads or in situations where the internet connection may be unstable or intermittent.

3. **Bandwidth Optimization:** With limited speed downloads, organizations can optimize their bandwidth allocation. By controlling the download speed, they can prioritize critical traffic, manage network congestion, and ensure a smoother experience for all users. Bandwidth optimization can be particularly beneficial for organizations with limited network resources or those serving a large number of concurrent users.

4. **Monetization Opportunities:** Limited speed downloads can be leveraged as a monetization strategy for websites. By offering faster download speeds as a premium feature, organizations can incentivize users to upgrade to a premium or paid membership. This approach allows websites to generate revenue and support the costs of providing high-speed downloads to their premium users.

5. **Download Management Compatibility:** By implementing pause/resume capability, the LimitedSpeedDownloadResult class ensures compatibility with a wide range of download managers and software. Some download managers treat downloads with limited concurrent sessions as one-time links, disabling pause/resume functionality. By bypassing temporary link restrictions, the solution maintains pause/resume functionality while balancing the download speed and preventing abuse.

6. **Security Enhancement:** Limited speed downloads can contribute to improved security for sensitive files. By controlling the download speed, organizations can minimize the risk of unauthorized access or data breaches. Slowing down the download process allows more time for security measures, such as authentication and authorization, to validate and monitor the file transfer.

## Usage

In order to implement this feature in your ASP.NET Core application, you don't have to use the raw code provided above. Instead, you can use the NuGet package `Matinm.Web.LimitedSpeedDownload`. As an example, you can install the latest version of this package in Visual Studio by typing this command in Package Manager Console:

```
NuGet\Install-Package Matinm.Web.LimitedSpeedDownload
```

for more details about installation, visit [Package's page in NuGet.org][https://www.nuget.org/packages/Matinm.Web.LimitedSpeedDownload/].

After you have the package installed, you can use the new IActionResult implementation provided by this package: `LimitedSpeedDownloadResult`. There are no limitations. You can use this package in both Razor Pages and MVC templates following samples below:

```csharp
public IActionResult OnGet ()
{
    // Example in Razor Pages OnGet method
	return new LimitedSpeedDownloadResult (@"C:\Users\Matin\Desktop\file.zip", 20, true);
}
```

```csharp
[Route ("/Download")]
[HttpGet]
public IActionResult Download ()
{
    // Example in MVC action
	return new LimitedSpeedDownloadResult (path, 50);
}
```

## Conclusion

In conclusion, implementing limited speed downloads with pause/resume capability in ASP.NET Core brings numerous benefits for websites and organizations. From controlled server load and enhanced user experience to bandwidth optimization and monetization opportunities, this approach provides a practical solution for managing file downloads effectively. Additionally, compatibility with download managers and security enhancements further strengthen the advantages of this implementation.
