---
title: Make an ASP.NET Core app's content localizable
author: wadepickett
description: Learn how to make an ASP.NET Core app's content localizable to prepare the app for localizing content into different languages and cultures.
ms.author: wpickett
monikerRange: '>= aspnetcore-5.0'
ms.date: 06/20/2025
uid: fundamentals/localization/make-content-localizable
---
# Make an ASP.NET Core app's content localizable

[!INCLUDE[](~/includes/not-latest-version.md)]

:::moniker range="> aspnetcore-5.0"

By [Hisham Bin Ateya](https://twitter.com/hishambinateya), [Damien Bowden](https://github.com/damienbod), [Bart Calixto](https://twitter.com/bartmax) and [Nadeem Afana](https://afana.me/)

One task for localizing an app is to wrap localizable content with code that facilitates replacing that content for different cultures.

## `IStringLocalizer`

<xref:Microsoft.Extensions.Localization.IStringLocalizer> and <xref:Microsoft.Extensions.Localization.IStringLocalizer%601> were architected to improve productivity when developing localized apps. `IStringLocalizer` uses the <xref:System.Resources.ResourceManager> and <xref:System.Resources.ResourceReader> to provide culture-specific resources at run time. The interface has an indexer and an `IEnumerable` for returning localized strings. `IStringLocalizer` doesn't require storing the default language strings in a resource file. You can develop an app targeted for localization and not need to create resource files early in development.  

The following code example shows how to wrap the string "About Title" for localization.

[!code-csharp[](~/fundamentals/localization/sample/8.x/Localization/Controllers/AboutController.cs)]

In the preceding code, the `IStringLocalizer<T>` implementation comes from [Dependency Injection](~/fundamentals/dependency-injection.md). If the localized value of "About Title" isn't found, then the indexer key is returned, that is, the string "About Title".

You can leave the default language literal strings in the app and wrap them in the localizer, so that you can focus on developing the app. You develop an app with your default language and prepare it for the localization step without first creating a default resource file.

Alternatively, you can use the traditional approach and provide a key to retrieve the default language string. For many developers, the new workflow of not having a default language *.resx* file and simply wrapping the string literals can reduce the overhead of localizing an app. Other developers prefer the traditional work flow as it can be easier to work with long string literals and easier to update localized strings.

## `IHtmlLocalizer`

Use the <xref:Microsoft.AspNetCore.Mvc.Localization.IHtmlLocalizer%601> implementation for resources that contain HTML. <xref:Microsoft.AspNetCore.Mvc.Localization.IHtmlLocalizer> HTML-encodes arguments that are formatted in the resource string, but doesn't HTML-encode the resource string itself. In the following highlighted code, only the value of the `name` parameter is HTML-encoded.

[!code-csharp[](~/fundamentals/localization/sample/8.x/Localization/Controllers/BookController.cs?highlight=3,5,20&start=1&end=24)]

***NOTE:*** Generally, only localize text, not HTML.

## `IStringLocalizerFactory`

At the lowest level, <xref:Microsoft.Extensions.Localization.IStringLocalizerFactory> can be retrieved from of [Dependency Injection](~/fundamentals/dependency-injection.md):

[!code-csharp[](~/fundamentals/localization/sample/8.x/Localization/Controllers/TestController.cs?highlight=6-12&name=snippet_1)]

The preceding code demonstrates each of the two factory create methods.

## Shared resources

You can partition your localized strings by controller or area, or have just one container. In the sample app, a marker class named `SharedResource` is used for shared resources. The marker class is never called:

[!code-csharp[](~/fundamentals/localization/sample/8.x/Localization/SharedResource.cs)]

In the following sample, the `InfoController` and the `SharedResource` localizers are used:

[!code-csharp[](~/fundamentals/localization/sample/8.x/Localization/Controllers/InfoController.cs?name=snippet_1)]

## View localization

The <xref:Microsoft.AspNetCore.Mvc.Localization.IViewLocalizer> service provides localized strings for a [view](xref:mvc/views/overview). The `ViewLocalizer` class implements this interface and finds the resource location from the view file path. The following code shows how to use the default implementation of `IViewLocalizer`:

[!code-cshtml[](~/fundamentals/localization/sample/8.x/Localization/Views/Home/About.cshtml)]

The default implementation of `IViewLocalizer` finds the resource file based on the view's file name. There's no option to use a global shared resource file. `ViewLocalizer` implements the localizer using `IHtmlLocalizer`, so Razor doesn't HTML-encode the localized string. You can parameterize resource strings, and `IViewLocalizer` HTML-encodes the parameters but not the resource string. Consider the following Razor markup:

```cshtml
@Localizer["<i>Hello</i> <b>{0}!</b>", UserManager.GetUserName(User)]
```

A French resource file could contain the following values:

| Key                        | Value                         |
| -------------------------- | ----------------------------- |
| `<i>Hello</i> <b>{0}!</b>` | `<i>Bonjour</i> <b>{0} !</b>` |

The rendered view would contain the HTML markup from the resource file.

Generally, ***only localize text***, not HTML.

To use a shared resource file in a view, inject `IHtmlLocalizer<T>`:

[!code-cshtml[](~/fundamentals/localization/sample/8.x/Localization/Views/Test/About.cshtml?highlight=5,12)]

## DataAnnotations localization

DataAnnotations error messages are localized with `IStringLocalizer<T>`. Using the option `ResourcesPath = "Resources"`, the error messages in `RegisterViewModel` can be stored in either of the following paths:

* *Resources/ViewModels.Account.RegisterViewModel.fr.resx*
* *Resources/ViewModels/Account/RegisterViewModel.fr.resx*

[!code-csharp[](~/fundamentals/localization/sample/8.x/Localization/ViewModels/Account/RegisterViewModel.cs)]

Non-validation attributes are localized.

<a name="one-resource-string-multiple-classes"></a>

### How to use one resource string for multiple classes

The following code shows how to use one resource string for validation attributes with multiple classes:

```csharp
    services.AddMvc()
        .AddDataAnnotationsLocalization(options => {
            options.DataAnnotationLocalizerProvider = (type, factory) =>
                factory.Create(typeof(SharedResource));
        });
```

In the preceding code, `SharedResource` is the class corresponding to the *.resx* file where the validation messages are stored. With this approach, DataAnnotations only uses `SharedResource`, rather than the resource for each class.

## Configure localization services

Localization services are configured in `Program.cs`:

[!code-csharp[](~/fundamentals/localization/sample/6.x/Localization/program.cs?name=snippet_LocalizationConfigurationServices)]

* <xref:Microsoft.Extensions.DependencyInjection.LocalizationServiceCollectionExtensions.AddLocalization%2A> adds the localization services to the services container, including implementations for `IStringLocalizer<T>` and `IStringLocalizerFactory`. The preceding code also sets the resources path to "Resources".

* <xref:Microsoft.Extensions.DependencyInjection.MvcLocalizationMvcBuilderExtensions.AddViewLocalization%2A> adds support for localized view files. In this sample, view localization is based on the view file suffix. For example "fr" in the `Index.fr.cshtml` file.

* <xref:Microsoft.Extensions.DependencyInjection.MvcDataAnnotationsMvcBuilderExtensions.AddDataAnnotationsLocalization%2A> adds support for localized `DataAnnotations` validation messages through `IStringLocalizer` abstractions.

[!INCLUDE[](~/includes/localization/currency.md)]

## Next steps

Localizing an app also involves the following tasks:

* [Provide localized resources for the languages and cultures the app supports](xref:fundamentals/localization/provide-resources)
* [Implement a strategy to select the language/culture for each request](xref:fundamentals/localization/select-language-culture)

## Additional resources

* [Url culture provider using middleware as filters in ASP.NET Core](https://andrewlock.net/url-culture-provider-using-middleware-as-mvc-filter-in-asp-net-core-1-1-0/)
* [Applying the RouteDataRequest CultureProvider globally with middleware as filters](https://andrewlock.net/applying-the-routedatarequest-cultureprovider-globally-with-middleware-as-filters/)
* <xref:fundamentals/localization>
* <xref:fundamentals/localization/provide-resources>
* <xref:fundamentals/localization/select-language-culture>
* <xref:fundamentals/troubleshoot-aspnet-core-localization>
* [Globalizing and localizing .NET applications](/dotnet/standard/globalization-localization/index)
* [Localization.StarterWeb project](https://github.com/aspnet/Entropy/tree/master/samples/Localization.StarterWeb) used in the article.
* [Resources in .resx Files](/dotnet/framework/resources/working-with-resx-files-programmatically)
* [Localization & Generics](http://hishambinateya.com/localization-and-generics)

:::moniker-end

[!INCLUDE [make-content-localizable5](~/fundamentals/localization/includes/make-content-localizable5.md)]
