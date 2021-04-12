---
title: A dotnet migration story
author: Manuele Lucchi
date: 2021-04-12 9:25:00 +0800
categories: [Projects, Libraries]
tags: [dotnet, csharp, netframework, wpf]
math: true
mermaid: true
---
For the last couple of months, one of my tasks was to gradually migrate 2 big WPF solutions from an old workflow to a more recent one. That includes upgrading the runtime version (from .NET Framework 4.5.2 to possibly .NET 5) but also the CI/CD and many other things. This article sums up the (not so bad) experience.

## What's there?
When I started digging on the projects there were 2 solutions (One for each product) with respectively 32 and 70 C# projects and that we'll call product __A__ and __B__ through the article. Most of them were libraries linked to the main executable (a WPF app) and some manual testing applications and basically all targeted .NET 4.5.2 (some 4.5). There were some references to COM components, OCX libraries and native DLLs.  

The solutions were structured in a pretty common way, with a folder for each layer: Presentation, Services, Data, Domain. Some of the projects of B where in common with A, but they had to manually copy the updated binaries. This set is internally called "Framework".

There was no unit testing and a limited CI/CD based on a really fragile environment. You couldn't just clone the repo and starting working having Visual Studio with the .NET tools installed, you had to install the [DevExpress workload](https://www.devexpress.com/) and copy many binaries into specific positions. I was working on the v2 of the softwares, while the v1 was still maintained and features added via __service packs__.

Also, due to a specific extension, the developers were stuck on Visual Studio 2017. That was annoying since I was using 2019 daily (that is better at everything) but the side by side installation solved the problem in no time. A little note: while I had 2017 installed, I used it only to try if everything compiles and run as a final check, but did everything else using 2019. This behavior resulted in a little problem later, so I suggest anyone in the same condition to just stuck with 2017 along the others.

## First steps
The first milestone I wanted to achieve was to enable _the future_.

.NET Framework 4.5.2 doesn't support .NET Standard 2.0 (the first really usable version of .NET Standard) that is the perfect bridge between .NET Framework and .NET Core. At least half of the projects of each solution contains code that is 99% CoreCLR compatibile, so the migration would not be problematic. But to have the presentation projects to reference .NET Standard 2.0 assemblies, at least .NET 4.6.x is needed. I ended up choosing 4.7.2. 

Here comes the first problem: I had to manually change the target framework for all the 100+ projects. And for each of them, there was a 2-3 seconds delay while Visual Studio was reloading the project freezing everything. It wasn't a pleasant experience. I even tried an extensions for project management, but it was so laggy to be unusable. I hope Microsoft adds a view to manage more projects properties at once in Visual Studio vNext. Even if the journey wasn't the best, there were no side effects on this upgrade, and everything continued to work smootly.

Another small issue was the need for the developers to install the .NET Core 2.0 SDK, that is deprecated in favor of the 2.1. But the 2.1 comes with .NET Standard 2.1, that isn't supported from the .NET Framework. So we just had to go along with it. 

All the __new__ projects not dependant on WPF or Windows only assemblies were now created targeting .NET Standard 2.0.

## External Packages
The second milestone was to have both products being able to run just by cloning them and having Visual Studio (and the correct SDK) installed. DevExpress was a major obstacle into this achievement, but since it's widely used into at least half of the projects I couldn't just toss it. The installation of the DevExpress package adds its DLLs into the "Extensions" section, so you can reference them. Updates were not so easy, since you had to install the new version and then use a tool that updates automatically the references (or in alternative, if you have time to spare, you can change them manually). This means tons of versions installed side by side and a long list of DLLs to choose from. 

So I made a short trip to their website and discovered they offered also Nuget packages using a private feed url. I just added a "nuget.config" with the feed url, removed the old references and installed the nuget counterpart (and I took advantage of it to update the version from 19 to 20) and, besides some APIs removed in the upgrade that we had to change, everything went smootly.

From now on, they can update every project at once with a click in the nuget explorer. They also gained a better theme management, since they had to add new themes manually, while now they are all included and updated in a single Nuget package.

With this and moving some binaries in better places, the projects now don't need anything besides Visual Studio to run. But it's still too much complex in some things, like manually copy new binaries versions to the versioning repository.

## Visual Studio 2017
We then started to port everything we could to Standard. It's not just a re-targeting, .NET Standard supports only the new SDK Style projects format. But Microsoft blessed use with [try-convert](https://github.com/dotnet/try-convert), a tool that converts old framework projects into .NET Standard (if they don't contain windows only code) or .NET 5 projects. It works pretty well and without any downside many projects were converted. 

After achieving so much in the Services/Data/Domain Layers, I moved my attention to the presentation. So, I tried to convert a WPF lib to the new SDK Style project, while maintaining 4.7.2 as target framework. Using try-convert, it converts the project to the new format, targeting .NET 5 and removing all the WPF direct references replacing them with `<useWpf>true</useWpf>`. This last tag and more generally the concept of "workloads" are really useful to keep everything clean and updated (you can use them with WPF, WinForms and soon Maui). I just had to retarget it back to .NET 4.7.2 (since they were not ready to move to .NET 5). Everything worked perfectly. _On Visual Studio 2019_.

That's right, `<useWpf>` used with .NET Framework and Visual Studio 2017 doesn't work. But it's not like it gives you an error, it just doesn't auto compile xaml files and link the correct assemblies. I even opened an [issue](https://github.com/dotnet/wpf/issues/4373), it seems like it's a SDK version issue. So, until the team moves to Visual Studio 2019, the presentation projects will remain _in the past_.

## Nuget Packages and multi targeting
Previously I said there were some projects shared between the two solutions, but only contained in solution B and used by A by copying the DLLs. 

That's of course not the greatest solution, so I put them in a separate solution (called __C__ from now on) and added each project as a nuget package in a private server. The procedure is extremely straightforward (you just need a nuget.config with the source you want to push the packages to and to check "generate nuget packages on build on the proj properties") and I just had to change from direct reference to the nuget. 

I took the initiative and ported these projects to Standard 2.0 (they were in the bottom layer, so they were almost only platform-agnostic code), but then a problem arises. The old version of the products (1.x) still targets NET 4.5.2 and they need to update the projects of C with changes valid for both versions (1.x and 2.x). Also, the 1.x project uses some third party nuget of a specific version, that have been updated in the 2.x. Thankfully, the new .NET project system is really awesome and allows multi-targeting, perfectly handling this really common scenario.


## IoC

## CI/CD

## Unit Testing