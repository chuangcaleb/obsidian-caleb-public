---
up: null
created: '2024-06-05T00:00:00.000Z'
modified: '2024-06-05T00:00:00.000Z'
collection:
  - meta
slug: meta/public-garden-navigation
---
# Public Garden Navigation

[🚧 WIP]

This note explains the navigational design of my public digital garden, and the different ways that users can explore the garden through note relations.

- [ ] explain garden exploration and connection
- [ ] gardens promote topography

The main mode of interaction is via the web page, which has been fitted with Obsidian-like navigational tools. But the web is just a frontend interface — the data content level itself must be designed to promote exploration.

## Directory

The strictest mode of topography is the (file) directory.

One folder can have many notes, and each note can have only one (direct) parent folder. There’s a strict hierarchal relationship. From the note, it’s a One-to-Many relationship.

The URL (universal resource locator) is an in-built breadcrumb that exposes the hierarchy trail.

Content systems of old primarily (and even exclusively) use directories to organise their content.

- [ ] why is strict good / bad
- [ ] Directory index files
- [ ] notes to navigate up

For now, the blog will keep a flat one-level directory for simplicity on the coding side lol.

## Collections

- collections are like soft directories
	- except each note can have multiple collections
	- not tied to the directory, won’t show up in the URL
- as a subset, a series-collection works just like a collection, except that its notes are ordered serially — that is, numbered in sequence. For example, a series on…
	- the series collection needs to list in order
	- and the individual notes should have this order accessible in itself, to navigate prev/forward/up

## Tags

- Tags are actually like collections
- without a collection index page that holds some content of its own
- here is a tag index page, but it’s simpler, just showing the connections
- under the hood, there is no source file for tags, whereas collection pages has an actual source note
-  the tag notes are just identical empty template shells
- collections are usually narrower in categorisation, tags are usually broad-scoping
- later allowing nested tags?

## Alternative Relations

- the metadata of a note may link to other notes as well
- usually these are a rarer case-by-case scenario
- give examples

## Backlinks

- Backlinks are the most interesting relation, because they can be surprising.
- It’s a list of other notes that points to the current note
- Great for finding connections that you don’t expect

## History

- Finally, not technically designed, but the navigation history stack is technically a way to explore notes
- This is not designed on the content data sie but is just there on the client viewing side
- It’s a trail or chain relationship, where each note connects to forward and backward in a chain
- Usually it comes from the user clicking one 
- Andy Mustechsk’s sliding panes takes this concept all the way, with old notes still shown on the screen on the left, stil in physical and mental viewport. Since it’s very easy to “pop” back to previous notes, viewers are more inclined to “dive” into deep rabbit trails.
	- One notable difference is while normal application or web browser navigation history, is that when navigating to a page/note that exists somewhere below in the stack, 
	- they will just push a duplicate entry in the stack, and then keep pushing more items, never popping from it.
	- whereas sliding panes will pop all notes in between until it reaches that previous stack item of access, and continue from there. This means that there’s no duplicates in the sliding panes stack.
	- This does have the benefit that you really feel like you’re backtracking your “trail of thought”, rather than just running around only in a forward direction.
