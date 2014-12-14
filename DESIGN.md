# OpenPlayground Design Document

## Goals

* Easy to install and administer (anyone with a basic shared hosting account should be able to join in)
* Able to be deployed to pretty much any cheap shared webhosting
* Easy to use
* Easy to theme/personalize
* Scalability/reliability; Minimal database dependency
  * Use a static generation model to reduce runtime database dependency
  * Backend should work on anything that can do simple indexed key-value storage (SQLite, DynamoDB, bdb, etc.)
  * Work on a publication-queue model by default
* Secure and privacy-focused by default (e.g. all non-public data should assume non-authorized and require proof of auth)
* Anyone can join in; no lock-in on protocol. Use OpenID, RSS/Atom, and standard Web extensions for everything to work.

## Key components

The distributed social network needs the following components:

* Identification
* Authorization
* Publication
* Subscription (dashboard)
* Browser extensions

## Identification

All identification is simply provided via OpenID. The URL should be associated with one's own primary presence, or perhaps their profile page.  The profile page itself can be managed by the publication system, or it can be provided by the identity provider and simply provide basic information like name, websites, etc.

For a first pass, an existing provider such as SimpleID can be used, although for a turnkey solution it should be integrated into the distribution, and ideally tightly coupled into the publication engine.

## Authorization

The core authorization service should allow people to provide their identity, and then maps that identity to an internal "profile." That profile can then be assigned to permissions groups by the site owner.  The profile should also support multiple auth mechanisms (OpenID, Twitter/Tumblr/etc. OAuth, emailed single-login tokin) to be bound to the individual profile, so that subscribers can change their auth mechanism or support multiple things on different devices.

On the server side, it should provide an easily-queried library that, given a session ID, provides the profile link and the permissions groups the user is in.

It should also provide a random (and revokable) random token that can be used for persistent sessionless authentication, e.g. for an RSS reader.

New authorization requests, auth stats, etc. should go into an RSS feed for the admin to view (so that they know to grant permissions to people or whatever).

## Publication

The publication engine should go for as much of a static-publication thing as possible; however, this requires some dynamic scripting support (for authorization), so this either means emitting PHP directly (that then hooks into the authorization library), or having some sort of very simple request router that then serves up mostly-statically-generated rendered things with a per-perimssion-group sandbox.

The publication engine should allow for different page types to have different URL schemata, templates, etc.  The templating sytem should be based on something intended for static publication; I like Movable Type's templating system, but there are many open-source templating systems which may be more appropriate.

For example, blog entries should be able to map to something like /blog/category/id-title, /blog/year/month/id-title, etc., with appropriate previous/next links between them; webcomic entries should be able to map to /comic/series/id-title, /comic/year/month/day/id-title, etc., with previous/next links within the series or between comic entries or whatever, and the ability to show any associated news posts with the comic as well.

The user interface should make it easy for users to "reblog" content (i.e. import a snippet from elsewhere on the web and optionally comment on it, with attribution back to the original link).

There should also be provisions for queued (e.g. "post N items per day between these hours") and scheduled (e.g. "post this item at 12:00 PM on Sunday") content.

### Filtering

When an item is published, it should be very very simple to render out, and it should have all the data for things like previous/next links "baked in" for all the possible affected ACLs. For example, if item 5 is public, 6 is friends-only, 7 is family-only, 8 is friends-or-family, and 9 is public, the metadata should look like this for item 7:

* Display: only if user is in the "family" group
* Previous link: 6 if user is in "friends," 5 if otherwise
* Next link: 8 if user is in "friends" or "family," 9 otherwise

and for item 8:

* Display: only if user is in "friends" or "family"
* Previous link: 7 if user is in "family", 6 if user is in "friends", 5 otherwise
* Next link: 9

Again, to keep this system scalable, the actual rendered content would be part of these itemsm (with all of the filtering logic baked into them), and it would be either the request router or the page itself (using simple embedded PHP) that would make these decisions, and if it's unable to contact the auth server, the assumption would be that the ACL group is "none."

If display is limited by an ACL, there should also be a `<meta>` tag on the page to reflect that, which would disable (or at least put up a warning for) simple reblogging functionality. (Obviously nothing stops people from copy-pasting the entry itself, but that's no different than any other social network.)

### Formatting

The "lingua Franca" of the publication engine should be something simple to use but which doesn't get in the way of using advanced HTML features; for example, Markdown or MediaWiki syntax or the like.  It should also be able to drop into pure HTML.  There should also be a WYSIWYG editor that makes it easy for people to edit without learning any markup, and this should also support drag-and-drop of assets that automatically upload and then inline-insert the asset into the item.

### Assets

Asset management should be set up in a way that people can upload assets to any sort of backing storage that they want; generally, uploading a file should just generate a SHA hash of the content that then maps onto the asset store at render time. This asset store could be then backed by a basic file store or an S3 bucket or the like.  Using the original content hash also solves the problem of asset invalidation and so on.

Whatever fronts the asset retrieval should be able to transcode (and cache) things as appropriate; images should be able to be fetched at various resolutions (driven by the layout template), audio should be transcodable to wav/ogg/flac/etc., animated gifs should be convertible to WebM/m4v/etc., and so on.

The asset retrieval service should also be set up such that it can be trivially fronted by a caching CDN, so that high-traffic users have the option of using e.g. CloudFront or [Coral CDN](http://coralcdn.org/) or the like.

### Metrics

The pages should also be able to log to a very simple analytics engine; this should keep track of page views with originating IP address, HTTP referer, viewer identity, and so on, and this should be made visible to the site owner. There will of course need to be some basic anti-spam measures (e.g. only reporting referers that come from multiple IP addresses and aren't just from the root page of a site or the like).  Daily/weekly/monthly reports would be useful as well.

### Comments

Pages should be able to accept comments directly on them (if so selected by the owner of the site). The comment engine doesn't have to be that fancy from the posters' perspective; just supporting a minimal subset of Markdown should be enough. The site owner should be able to limit commenting to specific auth groups.

There should also be the ability for people to "like" an item, which would simply add their viewer profile to the list of "likes" for that page. (Or it could just post a stub comment, similar to what Tumblr does.)

### Publicaton queue

There should be a queue of items which need to be updated; when an item is edited, commented on, scheduled, etc. it should go into the publication queue.  Any action that could trigger a render should also immediately start a publication worker up, so that the results of the publication become available as soon as there are sufficient server resources to have completed this publication (e.g. someone posting a comment would republish the page it's on, but could also cause another page to re-render if it was scheduled for posting by now).

The user should of course also have a cron job that pumps the publication queue, but it shouldn't have to run that often; it's mostly to handle scheduled posts.

Using this model means there are more server resources in use when something on the site is changing, but fewer in use during the 99.99% case of the site simply existing (and thus allows for far more readers to concurrently access the site).

### RSS feeds

There should be the capability of people creating their own RSS feed that draws from whichever site section(s) they want. This RSS feed should also have the sessionless auth token.

If the site sees someone authorized but their auth token hasn't been used to fetch a feed in a while, it should display a notification to the reader reminding them that they can subscribe, and provide a link to the feed customizer.

There should always be a `<link rel="alternate">` feed link that goes to the "everything" link, with the current auth token appended, if any. There should also probably be one that's specific to the current site section, as well.

## Subscription

The subscription engine should just be an RSS/Atom reader.  A basic implementation would just fetch all the subscribed feeds and then provide a river of content, sorted by time.  A more interesting implementation would provide an "inbox" view with the ability to mark items read, increase/decrease the priority of certain kinds of items (based on user-specified filters or based on machine learning), and so on.

The feed fetching should be based on dynamically-adjusting intervals, similar to the dynamic update support added to [Feed On Feeds](https://github.com/plaidfluff/Feed-on-Feeds).

There should also be provisions for subscribing to non-RSS/Atom content sources; for example, being able to fetch someone's Tumblr or Facebook dashboards via OAuth, shared calendars via CalDAV or iCal, or site-specific features such as subscribing to phpBB forums or the like.  It should be easy to write content-fetching plugins.

Calendar-type entries (or RSS/Atom entries including a .ics enclosure or the like) should be re-exportable via an iCal feed.

[PubSubHubBub](https://code.google.com/p/pubsubhubbub/) support would be great, as well.

Items should of course have a quick link back to the publication engine's "reblog" mechanism.

## Browser extensions

Websites should offer a simple means of logging in, via a `<meta>` tag that states that an OpenID login would be accepted, and where to send the credentials.

Browser extensions can then store the user's OpenID URL, and if this `<meta>` tag exists, prompt the user for whether to provide that identity.

A browser extension can also see that there's an RSS/Atom URL and then forward that along to the dashboard subscription thing. (This isn't really any different than existing mechanisms for RSS discovery, but it's worth pointing out here, especially since most modern browsers have eschewed built-in RSS discovery.)

There should also be an extension for posting an item from another page on the Internet (i.e. "reblogging").  An example UI for that would be selecting a portion of the page to quote, and then clicking on a "share" button.

These extensions could also be implemented as bookmarklets.