---
title: "Piazza Integration for Slack"
date: 2019-01-16T00:49:17-04:00
author: Ze Xuan Ong
feature_image: /unofficial-piazza-integration-slack/feature.png
draft: false
summary: "The homeworks haven't come in for grading yet. Students complain that we don't respond to Piazza questions quickly enough. I suppose if the questions fly directly into our Slack, that might speed things up a tad. Piazza doesn't officially support an API, so I built around it!"
---

The homeworks are not forthcoming at the moment so I've taken the night off to write a Piazza Slackbot. You can find it [here](https://github.com/ongzexuan/piazza_slackbot), feel free to fork it for your use.

## Motivation

Our TA team for the Natural Language Processing course has a new brand new Slack channel. It would be great if we could collect all the relevant course information in one place so we wouldn't have to check so many places for notifications. This would include Github commit notifications (for the main course repo) and notifications for new Piazza posts. Ideally, we would also at some point have a monitoring bot for Gradescope and Autolab to provide daily logs of submission numbers, scores and job times, but let's start with the Piazza posts.

{{< figure src="/unofficial-piazza-integration-slack/poc.png" caption="Proof of concept!" >}}

## Building

As usual, I hoped that someone would have already done it. There is as of yet an official Piazza integration for Slack, so we'd need a custom one. There are a number of scripts floating around on the Internet for a Piazza Slackbot. I decided to adapt mine from [t-davidson's piazza-slackbot](https://github.com/t-davidson/piazza-slackbot).

### Connections

There are already established packages for managing the Slack and Piazza connections. In this case we used the unofficial `piazza-api` and `slacker`.

### Pulling the latest posts

The unofficial api does not provide for a means to pull the latest/new posts. One could in theory get the post count or something, and then pull the entire list of posts from Piazza. This is quite a costly request and Piazza will probably hate me.

Another method is to keep a record of posts already forwarded to Slack so we don't end up forwarding it again. This could take the form of an environment variable which keeps track of the post id of the last forwarded post. This method however would require conscientious attention in updating the environment variable per deployment.

Instead, a nice workaround of sorts is to make use of the feed feature, an endpoint which is called when the Piazza page is loaded. The feed provides information to populate the standard Piazza page. Instead of providing the full post information, it instead provides a short snippet of each post.

The feed is orderd in a priority queue sort of way. Pinned posts are at the top, and then the remaining posts follow a chronological order. It happens that the post numbers increase linearly, so as long as we can get past the pinned posts, the first non-pinned post will be the latest post. We can distinguish a pinned post from a non-pinned post by the presence of the `pinned` key in the dictionary that represents the post in the feed (Ideally we would use the value but it seems to be designed this way currently). With this, we can write a function that returns us the latest post id.

```
def get_max_id(feed):
    for post in feed:
        if "pin" not in post:
            return post["nr"]
    return -1
```

### Short Logic

With this function, the polling logic can then be simply:

1. At the start-up of the app, fetch the latest post id. This serves as the 'last served' post.
2. Each minute or interval, fetch the new latest post id. If there is a difference between the old and new latest ids, then we know new posts have occurred.
3. We can then walk over the ids between the old and new latest and forward their information to Slack. In this case, we did a full fetch of the post information so we can forward more information to Slack instead of just the small snippet.

At this point, we should also handle the case where there are missing posts. Although Piazza posts ids are created in increments of 1, deletions can sometimes occur (like when an admin deletes a post).

## Deployment

I deployed this bot on a free-tier Heroku instance. Was rather quick and hassle free.

For Heroku deployment some additional configuration files are needed. Specifcally the Procfile:

```
worker: python piazza_bot.py
```

Indicating the process type as `worker` instead of the more common `web` lets Heroku know that this app will not be binding to the HTTP port. This prevents Heroku from terminating the app prematurely.

## Conclusion

I'm just happy it still runs in the morning. In future I'd make it a more full-fledged bot where you can at least poll it for its health too.