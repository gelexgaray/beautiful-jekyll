---
layout: post
title: "The weird IgnoreRoute syntax"
date: 2018-01-31 00:00:00 +0200
comments: true
tags: ["ASP.NET", "ClickOnce", "Routes"]
publish: true
---
Last day I was working to deploy a ClickOnce application from an ASP.NET MVC website. Nothing was working, and looking for a solution, I arrived to a method [documented like this](https://msdn.microsoft.com/en-us/library/dd470170(v=vs.118).aspx) on the MSDN. Wow... a method to change ASP.Net MVC routes with expressions based on a syntax that nobody knows!

# The problem behind the problem
> DISCLAIMER: I'm new to MVC. Probably this post will contain mistakes. 

Let's me explain what was my problem step by step:

* On ASP.Net MVC, the framework by default handles any request.
* ClickOnce uses its own handlers.
* When ClickOnce gets behind MVC, it gets "hidden" by ASP.Net MVC handlers.
* You have to instruct ASP.Net MVC to ignore some requests in order to let them be handled by ClickOnce.

This is the way I arrived at the IgnoreRoute method. I wanted to instruct MVC to ignore any request for a ClickOnce handled URL.

## ClickOnce URLs

ClickOnce uses three extensions:

* **.application**: This is the entry point to the application 
* **.deploy**: Depending on the configuration, ClickOnce deploys applications files using their original extension, or files renamed withe .deploy extension. So, this extension refers to application files
* **.manifest**: Manifest with metadata

I wanted MVC to ignore any request with this extensions

# Back to the IgnoreRoute syntax

IgnoreRoute is an extension method applied to a RouteCollection that receives two parameters:

* *url* (System.String): The URL pattern for the route to ignore.
* *constraints* \[optional\] (System.Object): A set of expressions that specify values for the url parameter.

You could expect the url to be some kind of regular expression, but, as you can see [here](https://msdn.microsoft.com/en-us/library/cc668201.aspx), it uses its own syntax.

There is almost no reference to what "constrainst" means.

# The copy-paste syndrome

If you search for documentation on this topic, google will make you walk in circles around this examples:
* [Ignore Route in MVC](http://www.c-sharpcorner.com/UploadFile/f82e9a/ignore-route-in-mvc/)
* [Make Routing Ignore Requests For A File Extension](https://haacked.com/archive/2008/07/14/make-routing-ignore-requests-for-a-file-extension.aspx/)

So, everybody is using two examples:
* One to ignore a file or folder
* The other one to ignore certain extension

Well... it was enough for me, but I was feeling like if nobody really has the knowledge about the real syntax of this method.

# Researching on the syntax

Viewing examples, I was sure the "constraints" was a collection of regular expressions. I found one page describing the "url" syntax:
* [ASP.NET Routing](https://msdn.microsoft.com/en-us/library/cc668201.aspx)

# My guess about how this stuff works
## First parameter (url)
Well, here comes my conclussion:
* With the first parameter, you can split the url based on the slash character. You can use literals or your own group identifiers. The last group can have a wildcard (*) character, that expands this group among the rest of the slash characters.

Let's suppose you have the following URL: path/to/mypage

* path/to/{page}: Captures "mypage" into a group with the name "page"
* {app}/{*operation}: Captures "path" into "app", and "to/mypage" into "operation"
* {*url}: Captures "path/to/mypage" into "url"

## Second parameter (constraints)
The second parameter is an object composed of regular expression string. Each regular expression is mapped to the group with the same name captured in the url.

* {app = "^ab.*", operation = ".*cd$}: In the previous examples, tells that app must start with "ab" and  operation must end with cd.

## Example: Filter ClickOnce urls using a single regular expression

I think this is too complex, and a single regular expression should be enough to filter any url... so here comes the way to filter any url using a single regular expression:


```
routes.IgnoreRoute("{*url}", new { url = @"(.*)\.application(\?.*)?$" });
routes.IgnoreRoute("{*url}", new { url = @"(.*)\.manifest$" });
routes.IgnoreRoute("{*url}", new { url = @"(.*)\.deploy$" });
```

And this way we arrive to the end of the post. See you and Happy Coding!
