# PHOENIX 360 WEB APPS WHITE PAPER

In a nutshell the Phoenix 360 Web Apps is a methodology whereby several web apps can run under the same website or as separated websites at the same time, while sharing the same sessions and data.


## The Motivation Behind It

The [Elixir](https://elixir-lang.org/) programming language encourages us to break our code into applications that can be organized logically by context, just like the concept of micro-services promotes, but in Elixir we are dis-encouraged from keeping this application in separated repos, because the runtime supports natively the concept of applications.

Elixir as the concept of [umbrella projects](https://elixir-lang.org/getting-started/mix-otp/dependencies-and-umbrella-projects.html) that allows to break our code into applications, under the same repo, that at runtime will be started as independent applications under different Supervision trees, but this approach is not popular among everyone, and one of the main issues is related with controlling how configuration is applied, like it's order or if you want to apply it or not for some of the umbrella applications, while other complain about the mono-repo becoming to big and without proper separation of contexts.

In the [Elixir for Programmers](https://codestool.coding-gnome.com/courses/elixir-for-programmers) video course, by [Dave Thomas](https://en.wikipedia.org/wiki/Dave_Thomas_(programmer), we are introduced to dependency applications, that also have a well defined context and are standalone deployable, but contrary to umbrella applications, they live in separated repos, thus creating a clear line of separation between applications that do not tend to blur/blend as the project grows. In the Elixir community for embedded Elixir projects with [Nerves](https://www.nerves-project.org/) this approach is also known as [Poncho Projects](https://embedded-elixir.com/post/2017-05-19-poncho-projects/).

The Phoenix 360 Web Apps concept is inspired by Dave Thomas approach and from the need to build several web apps, that have to be deployed all under a single website, while at same time having each one as a separated website.

The word *Phoenix* comes form the fact that it uses the [Phoenix Framework](https://www.phoenixframework.org/) for the Elixir programming language as the building block for each 360 web app. The word `360` comes from the fact that one Phoenix web app will include any number of standalone web apps that need to be aggregated under the same website.

[Home](/README.md)


## An Use Case Example

Lets imagine we have the website `app.local` where we want to provide the resources *Links* and *Notes*, while at same time we want to provide each of them as a separated website.

So, each resource is a self contained and standalone web app that can be accessed as:

* links.local
* notes.local

Now, for having all the resources under the same website, each web app will be accessed as:

* app.local/links
* app.local/notes

Due to the nature of the Elixir runtime all this three websites run independently from an outside perspective, while in fact all of them are deployed as a single project, that is started from the `app.local` under separated Supervision trees. The web server for `app.local` will be the only one running and accepting requests for all the three websites on the same http port `80` and https port `443`. Technically we could also start, from `app.local`, the web servers for `links.local` and `notes.local`, but they would need to run in different ports.

Independently how the user accesses any of the above web apps, the user can always use the same account that he may have created through `app.local`, through `links.local` or through `notes.local`.

Also, the data for each of the three web apps is always in sync. For example, the server handling the requests for `app.local/links` is the exact same one that handles the requests for `links.local`.

[Home](/README.md)
