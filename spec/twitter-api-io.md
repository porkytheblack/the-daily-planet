Here are your focused docs
twitterapi.io — Search & Profile Tweet Docs
Base URL: https://api.twitterapi.io
Auth: All requests require the header X-API-Key: YOUR_API_KEY
Pricing: $0.15 per 1,000 tweets returned
Use Case 1 — Keyword / Topic Search
Endpoint: GET /twitter/tweet/advanced_search
This works exactly like Twitter's search bar — you pass a query string using Twitter's search syntax.
Parameters:
ParameterTypeRequiredDescriptionquerystring✅The search query (see syntax below)queryTypeenum✅Latest (chronological) or Top (most popular)cursorstring❌For pagination — leave empty for first pageQuery syntax examples:What you wantQuery string------Single keywordAIMultiple keywords (OR)"AI" OR "machine learning"Exact phrase"artificial intelligence"From a specific userfrom:elonmuskKeyword + from user"AI" from:elonmuskSince a date/timefrom:elonmusk since:2026-02-03_10:37:00_UTCDate rangeAI since:2026-02-01 until:2026-02-03Exclude retweetsAI -filter:retweetsOnly with linksAI filter:linksMinimum likesAI min_faves:100

Full syntax reference: https://github.com/igorbrigadir/twitter-advanced-search
Example request — latest tweets about "AI":

bashcurl --request GET \\
  --url 'https://api.twitterapi.io/twitter/tweet/advanced_search?query=AI&queryType=Latest' \\
  --header 'X-API-Key: YOUR_API_KEY'
Example request — Elon's tweets in the last hour:
bashcurl --request GET \\
  --url 'https://api.twitterapi.io/twitter/tweet/advanced_search?query=from%3Aelonmusk%20since%3A2026-02-03_10%3A37%3A00_UTC&queryType=Latest' \\
  --header 'X-API-Key: YOUR_API_KEY'
(Note: URL-encode the query — spaces → %20, : → %3A)
Response fields (key ones):
json{
  "tweets": [
    {
      "id": "...",
      "url": "...",
      "text": "...",
      "createdAt": "...",
      "likeCount": 123,
      "retweetCount": 123,
      "replyCount": 123,
      "viewCount": 123,
      "quoteCount": 123,
      "author": {
        "userName": "...",
        "name": "...",
        "isBlueVerified": true
      }
    }
  ],
  "has_next_page": true,
  "next_cursor": "..."
}
Each page returns up to 20 tweets. If has_next_page is true, pass next_cursor as the cursor param for the next page.
Use Case 2 — All Tweets from a Profile
Endpoint: GET /twitter/user/last_tweets
Fetches a user's most recent tweets in reverse chronological order (newest first). Up to 20 per page.
Parameters:
ParameterTypeRequiredDefaultDescriptionuserNamestring✅*—Twitter handle, e.g. elonmuskuserIdstring✅*—Numeric user ID — preferred, faster & more stableincludeRepliesboolean❌falseSet to true to include replies in resultscursorstring❌""For pagination — leave empty for first page*Provide either userName or userId, not both. If both given, userId wins.Example request — latest tweets from @elonmusk:
bashcurl --request GET \\
  --url 'https://api.twitterapi.io/twitter/user/last_tweets?userName=elonmusk' \\
  --header 'X-API-Key: YOUR_API_KEY'
Example request — with replies included:
bashcurl --request GET \\
  --url 'https://api.twitterapi.io/twitter/user/last_tweets?userName=elonmusk&includeReplies=true' \\
  --header 'X-API-Key: YOUR_API_KEY'
Response fields (key ones):
json{
  "tweets": [
    {
      "id": "...",
      "url": "...",
      "text": "...",
      "createdAt": "...",
      "likeCount": 123,
      "retweetCount": 123,
      "replyCount": 123,
      "viewCount": 123,
      "isReply": false,
      "author": {
        "userName": "elonmusk",
        "name": "Elon Musk",
        "isBlueVerified": true,
        "followers": 123456789
      }
    }
  ],
  "has_next_page": true,
  "next_cursor": "...",
  "status": "success"
}
```
> **Tip for "last hour" filtering:** This endpoint doesn't have a built-in time filter. Either use **Use Case 1** with `from:elonmusk since:...` for clean time-bounded results, or fetch pages from this endpoint and stop when `createdAt` is older than your target time.
---
## Pagination (both endpoints)
```
Page 1: no cursor param needed  →  get next_cursor from response
Page 2: cursor=<value from page 1>  →  get next_cursor from response
Page 3: cursor=<value from page 2>  →  ...and so on
Stop when: has_next_page === false