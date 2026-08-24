# Travlr Getaways

**CS-465: Full Stack Development I**

A full stack MEAN application built for CS-465. The customer facing site is server rendered with Express and Handlebars, the admin side is an Angular single page application, and both are backed by a shared RESTful API over MongoDB with JWT protected routes.

**Stack:** MongoDB / Mongoose, Express, Angular, Node.js
**Structure:** `app_server` (Handlebars customer site), `app_api` (REST API), `app_admin` (Angular SPA)

## Architecture

This project ended up using three different approaches to the front end, and putting them side by side made the tradeoffs obvious.

The customer facing site runs on Express with Handlebars templates. The server assembles the HTML on every request and ships a finished page to the browser. That is great for a public travel site: the first paint is fast, there is no framework to download before anything appears, and search engines see real content. The cost is that every click is a round trip, and the server is doing layout work it does not strictly need to do.

Plain JavaScript sat on top of those rendered pages for small bits of interactivity. It is quick to add and has no build step, but it does not scale well. Once you are managing state by hand across a bunch of scripts, you are writing a framework badly instead of using one.

The admin side is where the Angular SPA earned its place. The shell loads once, routing happens in the browser, and components request JSON from the API instead of asking for new pages. Navigation feels instant, application state like the logged in user and the auth token survives between views, and the components are reusable. The tradeoffs are a heavier initial bundle, a real build pipeline, and a hard dependency on the API existing and behaving. For an internal admin tool used by a handful of people all day, that is a trade worth making.

The backend uses MongoDB because a trip is naturally a document, not a set of joined rows. A trip has a code, a name, an image, a description, and pricing, and all of that lives in one place and comes back in one read. Mongoose provided enough schema to keep the data honest without forcing a rigid table structure, so adding a field to the trip model did not mean writing a migration.

The bigger reason is that the whole stack speaks JSON. A document comes out of Mongo as something very close to a JavaScript object, the API serializes it, and Angular consumes it. The data never has to be translated into a different shape as it crosses a boundary, which removes an entire category of mapping code and the bugs that come with it.

## Functionality

JSON and JavaScript get conflated constantly, but JSON is a data format, not a language. It borrowed the look of JavaScript object literals and stopped there. There are no functions, no comments, no variables, no `undefined`, and keys have to be double quoted strings. JavaScript is the thing that runs; JSON is the thing that gets sent.

That distinction is exactly why it works as the contract between tiers. Mongoose hands the controller a document, the controller returns it as JSON, Angular parses it back into a typed model, and Handlebars renders it on the server side. Every layer agrees on the shape of the data without agreeing on anything else, which is what lets the Angular admin app and the Express customer site consume the same endpoints.

There were a few refactors along the way that made the app noticeably better. The largest was pulling trip data out of the hard coded JSON file and rewriting the Express travel controller to fetch from the API instead. That gave one source of truth. When the API changed, both faces of the application changed with it, and the customer site stopped carrying a stale copy of the catalog.

On the Angular side, the HTTP calls got centralized into a trip data service so no component builds its own URLs, and the auth token got moved into an interceptor. Before that, every request would have needed to remember to attach its own `Authorization` header, which is the kind of thing that works until the one place you forget.

Reusable UI components paid off in the same way. The trip card is defined once, so changing the markup or the styling changes it everywhere, the listing stays visually consistent by default, and building a new view is mostly assembly rather than authoring. It also makes testing easier, since you are verifying one component instead of five copies that have quietly drifted apart.

## Testing

An endpoint is a URL that exposes a resource, and a method is the verb that says what you want done to it. In this project `/api/trips` and `/api/trips/:tripCode` are the endpoints, and `GET`, `POST`, `PUT`, and `DELETE` map onto read, create, update, and delete. Same resource, different intent, and the method is what carries that intent.

Testing them in Postman meant sending each verb and checking two things: the status code and the body. `GET` a list and confirm a 200 with an array. `POST` a trip and confirm it comes back with the fields you sent. `GET` a code that does not exist and confirm a 404 rather than an empty 200, since a lying status code is worse than an error.

Security is what makes this harder. Once routes are protected, a request is no longer just a URL and a body. You have to register or log in first, capture the JWT that comes back, and attach it as a bearer token on every subsequent call. If you skip that step, the endpoint you thought was broken is actually working correctly and rejecting you.

The negative cases matter more than the happy path here. A protected route needs to be tested with no token, with a malformed token, and with a token that has been tampered with, and it needs to return 401 in all three cases without leaking data. That last one is not theoretical. While implementing the auth middleware I hit a case where the verification callback ran asynchronously and the handler continued anyway, which meant an invalid token could reach the route. It only showed up because I tested the failure path instead of assuming the success path proved it worked.

A second bug worth recording: the DELETE endpoint existed and looked correct, but requests to it returned a 404. The cause was a duplicate `router.route('/trips/:tripCode')` block registered earlier in the file with only `.get` and `.put` attached. Express matches the first registered route for a path and stops, so the second block holding the delete handler was never reached. Removing the shadowing duplicate fixed it. Nothing about reading the delete handler in isolation would have revealed that, which is a good argument for testing routes through the running server rather than by inspection.

## Reflection

I am a QA analyst with about six and a half years of experience, and this course was aimed squarely at the transition into software engineering. What it gave me was the ability to work on both sides of a boundary I have spent my career testing from the outside. I can now build the API, protect it, consume it from a SPA, and explain why each piece is shaped the way it is.

The concrete skills are the MEAN stack end to end: Express routing and MVC structure, Mongoose schemas and MongoDB modeling, RESTful API design, Angular components, services, routing and interceptors, and JWT authentication with hashing and salting. Alongside that, working through the project on feature branches and committing in reviewable chunks got me used to a workflow that resembles the one I would join.

The part I did not expect to value as much as I do is the overlap with the work I already do. Debugging the auth flow, tracing why a request failed, and reading the course guide's own code critically enough to spot the places it was wrong all came straight out of the QA habit of not trusting that something works until it demonstrably does. That combination, someone who can build the feature and who instinctively goes looking for how it breaks, is the thing I want on a resume, and this project is the first portfolio piece that shows both at once.
