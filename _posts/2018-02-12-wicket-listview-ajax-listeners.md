---
layout: post
title:  "Wicket ListViews and Ajax listeners"
date:   2018-02-12 18:33:00 +0100
---
One of my favorite web frameworks is [Apache Wicket](https://wicket.apache.org/), a component-based server-side web
framework written in Java. I've been working with it almost daily since 2008 (though a bit less since switching jobs),
and it powers pretty much all of my personal web projects.

In one of those projects, Beholder, I ran into an interesting problem with regard to ListViews and Ajax listeners,
which I'll explain in this post.
<!--more-->
Consider the following HTML page:

```html
<!-- ListViewAndAjaxPage.html -->

<html xmlns:wicket="https://wicket.apache.org">
    <head>
        <title>ListView and Ajax</title>
    </head>
    <body>
        <div wicket:id="posts">
            <h2 wicket:id="title">Post title goes here</h2>
            <span wicket:id="preview">
                Post preview goes here
            </span>
            <a wicket:id="readmore">Read more</a>
        </div>
    </body>
</html>
```

With the following corresponding Wicket page:
```java
// ListViewAndAjaxPage.java

public class ListViewAndAjaxPage extends WebPage {
	public ListViewAndAjaxPage() {
		add(new ListView<Post>("posts", new PostsModel()) {
			@Override
			public void populateItem(Item<Post> item) {
				Post post = item.getModelObject();
                item.add(new Label("title", post.getTitle()));
                item.add(new Label("preview", post.getPreview()));
                item.add(new AjaxLink<Post>("readmore", item.getModel()) {
                	@Override
                	public void onClick(AjaxRequestTarget target) {
                		Post post = getModelObject();
                		displayPost(target, post);
                	}
                });
			}
		});
	}
	
	public void displayPost(AjaxRequestTarget target, Post post) {
		// Logic for displaying post without page refresh
	}
	
	public static class PostsModel extends LoadableDetachableModel<List<Post>> {
		@Override
		public List<Post> load() {
			return getPostsFromDatabase();
		}
	}
}
```

For those unfamiliar with Wicket, a small explanation:

* The `ListView` is bound to the HTML element with id `posts`, and is a repeater, which means the HTML will
be repeated for each element in the corresponding `PostModel`
* The children of the `ListView` are bound inside the `populateItem` method:
   * `title` is bound to a `Label` object
   * `preview` is bound to another `Label` object
   * `readmore` is bound to an `AjaxLink` object
   
When the page is loaded (assuming the `PostsModel` contains a list of 3 posts), the resulting page will look roughly like this:

```html
<html>
    <head>
        <title>ListView and Ajax</title>
    </head>
    <body>
        <div>
            <h2>Post A</h2>
            <span>
                Preview of post A
            </span>
            <a href="ajaxlink/handler/in/posts/item0/readmore">Read more</a>
        </div>
        <div>
            <h2>Post B</h2>
            <span>
                Preview of post B
            </span>
            <a href="ajaxlink/handler/in/posts/item1/readmore">Read more</a>
        </div>
        <div>
            <h2>Post C</h2>
            <span>
                Preview of post C
            </span>
            <a href="ajaxlink/handler/in/posts/item2/readmore">Read more</a>
        </div>
    </body>
</html>
```

The links I've shown aren't proper Wicket links, but they communicate the essence: when called, they'll
try to invoke the handler of the `AjaxLink` inside the `posts` component's `n`th item. Under normal circumstances,
this should work just fine, but there is 1 caveat.

## Item order

Between requests, the entire page is serialized, including all components and contained models. Models in Wicket
generally either hold a reference to something serializable *or* the logic to reconstruct something that is not
or not fully serializable.

The `PostsModel` above is a good example of this. It extends `LoadableDetachableModel`, which throws away
object references during the "detach" phase, and reconstructs them upon request through the `load()` method.

When the page is reconstructed from the serialized form, the `ListView` needs to be rebuilt. To do so, it
consults its model, which returns a list of posts. Depending on config, it will then either reconstruct
the items belonging to the list, *or* reuse them.

The problem arises when the list of posts has no defined order:

1. Page is created, with posts ordered A, B, C
2. Page is serialized
3. User clicks middle link, to consult item B (index 1)
4. Page is deserialized
5. `ListView` model reconstructed, but returns in order B,C,A
6. Request is deferred to click handler of item C

The problem is easy to solve: just ensure that the posts have a defined order, and you'll be fine, but debugging
this particular issue was interesting to say the least.
