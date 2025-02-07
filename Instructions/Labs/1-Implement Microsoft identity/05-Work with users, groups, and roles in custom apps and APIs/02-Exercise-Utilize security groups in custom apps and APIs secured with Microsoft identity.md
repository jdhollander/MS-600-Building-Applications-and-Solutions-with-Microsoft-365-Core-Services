# Exercise 2: Utilize security groups in custom apps and APIs secured with Microsoft identity

In this exercise, you’ll learn how to create and configure an application registration to include group claims in a token and authorize access to a controller using those claims.

> [!IMPORTANT]
> This exercise assumes you have completed the first exercise in this module. 

## Task 1: Update the application registration

The ID token returned by default from Microsoft identity contains only basic information about the current user. The application registration can be updated to include additional information. Configure the application registration to include group membership claims as role claims by editing the application manifest.

1. Open a browser and navigate to the [Azure Active Directory admin center](https://aad.portal.azure.com). Sign in using a **Work or School Account** that has global administrator rights to the tenant.

2. Select **Azure Active Directory** in the left-hand navigation.

3. Select **Manage > App registrations** in the left-hand navigation.

![Screenshot of the App registrations](../../Linked_Image_Files/01-05-azure-ad-portal-home.png)

4. On the **App registrations** page, locate the application registration that represents the Users Group Roles application from the first exercise in this module. To verify the application, compare the **Application (client) ID** and **Directory (tenant) ID** in the portal to the values copied in the first exercise.

![Screenshot of the application ID of the new app registration](../../Linked_Image_Files/01-05-03-azure-ad-portal-new-app-details-01.png)

5. In the application registration for your application, select **Manage > Manifest**.

6. In the manifest editor, find the node named `groupMembershipClaims`. The default value is `null`. Set the `groupMembershipClaims` node with the following code:

```json
"groupMembershipClaims": "SecurityGroup",
```

7. Next, within the manifest editor, find the node `optionalClaims`. The default value is `null`. Set the `optionalClaims` node to the following code:

```json
"optionalClaims": {
  "idToken": [
    {
      "name": "groups",
      "source": null,
      "essential": false,
      "additionalProperties": [
        "emit_as_roles"
      ]
    }
  ],
  "accessToken": [],
  "saml2Token": []
},
```

![Screenshot of the application registration with the manifest link highlighted](../../Linked_Image_Files/01-05-05-azure-ad-portal-appreg-manifest.png)

8. Save the manifest.

### Extend application with authorization

The Home controller of the application contains the `[Authorize]` attribute, which allows any logged-in user to view the page. A new controller will demonstrate using the group claims to authorize users.

The next step is to add a model, controller, and view to the web app that will display Products from a fictitious product catalog. Only members of a specific group are allowed to view products.

#### Create a product viewer group

9. Open a browser and navigate to the [Azure Active Directory admin center](https://aad.portal.azure.com). Sign in using a **Work or School Account** that has global administrator rights to the tenant.

10. Select **Azure Active Directory** in the left-hand navigation.

11. Select **Manage > Groups** in the left-hand navigation.

12. On the **All Groups** page, select **New Group**. Create the group with the following information:

- **Group Type**: Security
- **Group Name**: Product Viewers
- **Group Description**: Allows user to view products.
- **Membership type**: Assigned
- **Owners**: Select one user account as the group owner.
- **Members**: Select the same user account you selected as the group owner. You may optionally add additional accounts.

> [!NOTE]
> The user must be a member of the group to have it included in the group claim.

13. Select **Create**.

14. On the **All Groups** page, copy the **Object ID** of the new group. You'll need this value later in the exercise.

#### Add data models and sample data

The next step is to add data models and sample data to the web app.

15. In the **Models** folder, create a new file named **Category.cs** and add the follow C# code to it:

```csharp
namespace UserGroupRole.Models
{
  public class Category
  {
    public int ID { get; set; }
    public string Name { get; set; }
  }
}
```

16. In the **Models** folder, create a new file named **Product.cs** and add the following C# code to it:

```csharp
namespace UserGroupRole.Models
{
  public class Product
  {
    public int ID { get; set; }
    public string Name { get; set; }
    public Category Category { get; set; }
  }
}
```

This exercise will store sample data in-memory while the app is running. The data will randomly generated when the app is started using a NuGet package.

17. Install the NuGet package by running the following from your command prompt in the project folder:

```console
dotnet add package Bogus
```

18. Return to **Visual Studio Code** and create a new file named **SampleData.cs** in the root folder of the project. Add the following C# code to the file:

```csharp
using System.Collections.Generic;
using Bogus;
using UserGroupRole.Models;

namespace UserGroupRole
{
  public class SampleData
  {
    public List<Category> Categories { get; set; }
    public List<Product> Products { get; set; }

    public static SampleData Initialize()
    {
      var data = new SampleData();

      var categoryIds = 0;
      var categoryFaker = new Faker<Category>()
        .StrictMode(true)
        .RuleFor(c => c.ID, f => ++categoryIds)
        .RuleFor(c => c.Name, f => f.Commerce.Categories(1)[0]);
      data.Categories = categoryFaker.Generate(10);

      var productIds = 0;
      var productFaker = new Faker<Product>()
        .StrictMode(true)
        .RuleFor(p => p.ID, f => ++productIds)
        .RuleFor(p => p.Name, f => f.Commerce.Product())
        .RuleFor(p => p.Category, f => f.PickRandom(data.Categories));
      data.Products = productFaker.Generate(20);

      return data;
    }
  }
}
```

19. The sample data will be stored as a singleton in the dependency injection container built into ASP.NET Core. Open the **Program.cs** file in the root folder of the project. Add the following line immediately before this line of code `var app = builder.Build();`:

```csharp
builder.Services.AddSingleton(UserGroupRole.SampleData.Initialize());
```

### Add controller and view for Products

Earlier, the middleware was configured to use the group claim from Microsoft identity as the role claim in ASP.NET's identity system. The `[Authorize]` attribute has a property that can specify the roles required to access a controller. Since the group claim contains the group ID, the ID is specified in the attribute. The ID is the object ID copied from the Azure Active Directory admin center.

20. Add a new file **ProductsController.cs** to the **Controllers** folder. Add the following code to it:

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace UserGroupRole.Controllers
{
  [Authorize(Roles=("<VIEWER-GROUP-OBJECTID>"))]
  public class ProductsController : Controller
  {
    SampleData data;

    public ProductsController(SampleData data)
    {
      this.data = data;
    }

    public ActionResult Index()
    {
      return View(data.Products);
    }
  }
}
```

21. Replace the string `<VIEWER-GROUP-OBJECTID>` with the value copied from the All Groups page.

Now create the view to display the products.

22. Add a new folder **Products** to the **Views** folder. Add a new file, **Index.cshtml**, to the new **Products** folder and add the following code to it. This will display all the products provided by the API:

```html
@model IEnumerable<UserGroupRole.Models.Product>

@{
  ViewData["Title"] = "Products";
}

<h1>Products</h1>

<table class="table">
  <thead>
    <tr>
      <th>
        @Html.DisplayNameFor(model => model.ID)
      </th>
      <th>
        @Html.DisplayNameFor(model => model.Name)
      </th>
      <th>
        @Html.DisplayNameFor(model => model.Category)
      </th>
    </tr>
  </thead>
  <tbody>
@foreach (var item in Model) {
    <tr>
      <td>
        @Html.DisplayFor(modelItem => item.ID)
      </td>
      <td>
        @Html.DisplayFor(modelItem => item.Name)
      </td>
      <td>
        @Html.DisplayFor(modelItem => item.Category.Name)
      </td>
    </tr>
}
  </tbody>
</table>
```

The ASP.NET identity system allows for an imperative test of membership via the `User.IsInRole()` method. Use this method to update the site navigation, showing a link to the Products controller only if the user is allowed to access it.

23. Open the file **Views\Shared\\_Layout.cshtml**. In the `<header>` element is an unordered list (`<ul>`) of links that compose the navigation. The navigation has link to Home and Privacy. After the Privacy link, add the following code:

```cshtml
@if (User.IsInRole("<VIEWER-GROUP-OBJECTID>"))
{
  <li class="nav-item">
    <a class="nav-link text-dark" asp-area="" asp-controller="Products" asp-action="Index">Products</a>
  </li>
}
```

24. Replace the string `<VIEWER-GROUP-OBJECTID>` with the value copied from the All Groups page.

#### Build and test the web app

25. Execute the following commands in a command prompt to compile and run the application:

```console
dotnet dev-certs https --trust
dotnet build
dotnet run --urls https://localhost:5001
```

26. Open a browser and navigate to the url **https://localhost:5001**. The web application will redirect you to the Azure AD sign-in page.

27. Sign in using a Work and School account from your Azure AD directory. The login will prompt for consent to the scopes required by the web API if the account you are using has not previously consented. After login and consent, Azure AD will redirect you back to the web application.

> [!NOTE]
> You must login after adding users as members of the security group. Any logins that occurred before the users were added will result in tokens that does not reflect the membership. Close the browser or select **Sign out** to sign out of the session.

28. On the home page, the assigned groups are included in the list of claims as roles. If the user is a member of the correct group, the navigation will include a link to the Products controller.

![Screenshot of the application home page, highlighting the navigation and group claims.](../../Linked_Image_Files/01-05-05-home-page-with-groups.png)

## Summary

In this exercise, you learned how to create and configure an application registration to include group claims in a token and authorize access to a controller using those claims.
