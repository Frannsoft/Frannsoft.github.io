---
layout: post
title: "Hypermedia links via ActionFilters in MS Web Api 2"
date: 2017-07-08
tags: ms-web-api-2 c# rest
---

While rewriting a hobby API I maintain I've been learning a lot about the MS Web Api tooling and building good Restful APIs.

One of the core tenets of RESTful APIs is offering additional resources and actions within the response object structure via links embedded in the response data, a facet of [HATEOAS](https://en.wikipedia.org/wiki/HATEOAS) (Hypermedia). This way the consumer doesn't need to work so hard to find those resources themselves.

Here's a basic example using Ms Web Api:

```csharp
//Web Api Controller
public class MyController : ApiController {
    [Route ("/characters/{id}", Name = nameof (GetCharacter))]
    [HttpGet]
    public IHttpActionResult GetCharacter (int id) {
        var character = _service.GetCharacter (id);

        var characterDto = new CharacterDto {
            id = character.Id,
                Name = character.Name,
                Links = new [] {
                    new Link {
                        Rel = "self",
                            Href = $"/characters/{character.Id}"
                    },
                    new Link {
                        Rel = "moves",
                            Href = $"/characters/{character.Id}/moves"
                    }
                }
        }
        return Ok (characterDto);
    }
}

//Stores link information to get to other resources available from the
//returned response data.
public class Link {
    public string Href { get; set; }
    public string Rel { get; set; }
}

//Character model
public class Character {
    public int Id { get; set; }
    public string Name { get; set; }
}

//Character Dto
public class CharacterDto {
    public int Id { get; set; }
    public string Name { get; set; }
    public List Links { get; set; }
}
```

The caller will see the available actions off of this response object right inside the response object and therefore won't have to spend so much time searching documentation or other external info for these features. Another benefit is if the routes for these change, consumers will not need to fret as much since they can simply get the updated `Href` property off of the response via the `Rel` key for it. This way they don't need to worry about updating all the potential areas that consume the updated route directly.

However, look at how much code is added to this single example. Applying this to multiple Controller Actions is going to result in an annoying amount of boilerplate code.

Using Web Api's Action Filters you can do this in a bit more of an aspect-oriented way. Here's an example - Keep in mind this example does NOT use Asp.Net MVC, but strictly MS Web Api so I'm referencing the System.Web.Http.Filters namespace and not System.Web.Http.MvcFilters.

```csharp
public class MyController : ApiController {
    //our new ActionFilter will only execute on this specific Action giving us fine-grained control over when we want to add these Hypermedia links.
    [CharacterHypermediaResourceFilterAttribute]
    [Route ("/characters/{id}", Name = nameof (GetCharacter))]
    [HttpGet]
    public IHttpActionResult GetCharacter (int id) {
        var character = _service.GetCharacter (id);

        var characterDto = new CharacterDto {
            Id = character.Id,
            Name = character.Name
        };

        return Ok (characterDto);
    }
}

//Action Filter
public class CharacterHypermediaResourceFilterAttribute : ActionFilterAttribute {
    public override void OnActionExecuted (HttpActionExecutedContext actionExecutedContext) {
        base.OnActionExecuted (actionExecutedContext);

        //Get the existing response data we sent from the Controller action.
        CharacterDto responseContent;
        actionExecutedContext.Response.TryGetContentValue (out responseContent);

        //add the Hypermedia links we want.
        responseContent.Links = new Link {
            new Link {
                Rel = "self",

                    //we can use `nameof` for the Controller name in order to avoid having outdated routeNames by string in any future refactoring.
                    Href = actionExecutedContext.Request.GetUrlHelper ().Link (nameof (CharacterController.GetCharacter), responseContent.Id);
            }
        };
    }
}
```

This approach is fairly fine-grained. Our `CharacterHypermediaResourceFilterAttribute` could also be applied to the `CharacterController` class and every Action on the Controller would be affected. We could eliminate the manual mapping from model to DTO with [AutoMapper](https://github.com/AutoMapper/AutoMapper) as well, but that's outside the scope of this post.

In order to create the link successfully, you will need to set the `Name` property of the `RouteAttribute` you place on your controllers. Otherwise, the route name will not be able to be found when creating the link and an exception will be thrown.

If you are looking to apply Hypermedia links to many response objects (or potentially all of them) Ben Foster's custom Message Handler [post](http://benfoster.io/blog/generating-hypermedia-links-in-aspnet-web-api) is super useful.

Also, if you are not interested in creating a `Link` structure from scratch check out [WebApi.Hal](https://github.com/JakeGinnivan/WebApi.Hal) by Jake Ginnivan.