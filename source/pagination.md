---
title: Pagination
order: 21
---

Sometimes, you will have one or more views in your application where you need to display a list that contains too much data to be either fetched or displayed at once. Pagination is the most common solution to this problem, and Apollo Client has built-in functionality that makes it quite easy to do.

There are basically two ways of fetching paginated data: numbered pages, and cursors. There are also two ways for displaying paginated data: discrete pages, and infinite scrolling. For a more in-depth explanation of the difference and when you might want to use one vs. the other, we recommend that you read our blog post on the subject: [Understanding Pagination](https://medium.com/apollo-stack/understanding-pagination-rest-graphql-and-relay-b10f835549e7).

In this article, we'll cover the technical details of using Apollo to implement both approaches.


<h2 id="fetch-more">Using `fetchMore`</h2>

Apollo lets you do pagination with a function called [`fetchMore`](cache-updates.html#fetchMore), which provided on the `data` prop by the `graphql` HOC. You need to specify what query and variables to use for the update, and how to merge the new query result with the existing data on the client. How exactly you do that will determine what kind of pagination you are implementing.

<h2 id="numbered-pages">Offset-based</h2>

Offset based pagination - also called numbered pages - is a very common pattern, found on many websites, because it is usually the easiest to implement on the backend. In SQL for example, numbered pages can easily be generated by using [OFFSET and LIMIT](https://www.postgresql.org/docs/8.2/static/queries-limit.html).

Here is an example with numbered pages taken from [GitHunt](https://github.com/apollostack/GitHunt-React):

```js

const FEED_QUERY = gql`
  query Feed($type: FeedType!, $offset: Int, $limit: Int) {
    currentUser {
      login
    }
    feed(type: $type, offset: $offset, limit: $limit) {
      id
      # ...
    }
  }
`;

const ITEMS_PER_PAGE = 10;
const FeedWithData = graphql(FEED_QUERY, {
  options(props) {
    return {
      variables: {
        type: (
          props.params &&
          props.params.type &&
          props.params.type.toUpperCase()
        ) || 'TOP',
        offset: 0,
        limit: ITEMS_PER_PAGE,
      },
      forceFetch: true,
    };
  },
  props({ data: { loading, feed, currentUser, fetchMore } }) {
    return {
      loading,
      feed,
      currentUser,
      loadMoreEntries() {
        return fetchMore({
          // query: ... (you can specify a different query. FEED_QUERY is used by default)
          variables: {
            // We are able to figure out which offset to use because it matches
            // the feed length, but we could also use state, or the previous
            // variables to calculate this (see the cursor example below)
            offset: feed.length,
          },
          updateQuery: (previousResult, { fetchMoreResult }) => {
            if (!fetchMoreResult.data) { return prev; }
            return Object.assign({}, prev, {
              feed: [...previousResult.feed, ...fetchMoreResult.data.feed],
            });
          },
        });
      },
    };
  },
})(Feed);
```
[source code](https://github.com/apollostack/GitHunt-React/blob/c0b18795a18b3da42dc90cf7c63b29b14965206d/ui/Feed.js#L165)

As you can see, `fetchMore` is accessible through the `data` argument to the `props` function (or passed directly to a child component). As it makes sense to keep all data-specific logic together for your app, we use `props` to define a simpler "load more" function (`loadMoreEntries` in this case), to be used naively by the child component (`Feed` in this case).

In the example above, `loadMoreEntries` is a function which calls `fetchMore` with the length of the current feed as a variable. Whenever you don't pass a query argument to `fetchMore`, fetch more will use the original `query` again with new variables. Once the new data is returned from the server, the `updateQuery` function is used to merge it with the existing data, which will cause a re-render of your UI component.

In the example above, the `loadMoreEntries` function is called from the UI component:

```js
const Feed = () => {
  const { vote, loading, currentUser, feed, loadMoreEntries } = this.props;

  return (
    <div>
      <FeedContent
        entries={feed || []}
        currentUser={currentUser}
        onVote={vote}
      />
      <a onClick={loadMoreEntries}>Load more</a>
      {loading ? <Loading /> : null}
    </div>
  );
}
```

One downside of pagination with numbered pages or offsets is that an item can be skipped or returned twice when items are inserted into or removed from the list at the same time. That can be avoided with cursor-based pagination.

<h2 id="cursor-pages">Cursor-based</h2>

In cursor-based pagination a cursor is used to keep track of where in the data set the next items should be fetched from. Sometimes the cursor can be quite simple and just refer to the ID of the last object fetched, but in some cases - for example lists sorted according to some criteria - the cursor needs to encode the sorting criteria in addition to the ID of the last object fetched. Cursor-based pagination isn't all that different from offset-based pagination, but instead of using an absolute offset, it points to the last object fetched and contains information about the sort order used. Because it doesn't use an absolute offset, it is more suitable for frequently changing datasets than offset-based pagination.

In the example below, we use a `fetchMore` query to continuously load new comments, which then appear at the top. The cursor to be used in the `fetchMore` query is provided in the initial server response, and has to be updated whenever more data is fetched.

```js
const moreComments = gql`
  query moreComments($cursor: String) {
    moreComments(cursor: $cursor) {
      cursor
      comments {
        author
        text
      }
    }
  }
`;

const FeedWithData = graphql(FEED_QUERY, {
  props({ data: { loading, feed, currentUser, cursor, fetchMore } }) {
    // This technique only allows a single instance of `FeedWithData` in your app
    // A better approach would be to store the cursor on the component
    //   (however this is a little difficult right now:
    //    https://github.com/apollostack/react-apollo/issues/188)
    FeedWithData.cursor = cursor;
    return {
      loading,
      feed,
      currentUser,
      loadMoreEntries: () => {
        return fetchMore({
          query: moreComments,
          variables: {
            // cursor is the initial cursor returned by the original query
            // this.cursor is the cursor that we update via `updateQuery` below
            cursor: FeedWithData.cursor,
          },
          updateQuery: (previousResult, { fetchMoreResult, queryVariables }) => {
            const previousEntry = previousResult.entry;
            const newComments = fetchMoreResult.data.comments.nextComments;

            // update internal reference to cursor
            FeedWithData.cursor = fetchMoreResult.data.cursor;

            return {
              title: previousEntry.title,
              author: previousEntry.author,
              // put promoted comments in front
              comments: [...newComments, ...previousEntry.comments],
            };
          },
        });
      },
    };
  },
})(Feed);
```