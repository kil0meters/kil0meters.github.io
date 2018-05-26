---
layout: post
title:  "What I Learned from Building a Reddit Bot"
date:   2018-05-26 10:44:05 -0700
categories: reddit python
---

This project started out as many of my others do: *I have a problem there's currently no good solution to*.

In this case, my friend wanted to be able to get notifications when a post was made on certain subreddits. This was for subreddits like [/r/buildapcsales][r_buildapcsales] or [/r/hardwareswap][r_hardwareswap] where arriving to a thread early could be **vital** to getting a good deal. 

There were existing solutions such as [GRedditNotifier][GRedditNotifier], but after looking into them I quickly found they were far from ideal. In this case you were required to download pusbullet. I mean, who wants to download a completely separate app for a feature that should just be built into Reddit anyway.

Just like that, I began creating my own solution.

## Beginning

I had a pretty simple goal, so [reaching that goal][beginning] wasn't very difficult. I wrote the script in about an hour. 

Some interesting things to note are that I didn't use the Reddit API at all and instead abused RSS feeds such as `https://reddit.com/r/<subreddit>/new/.rss`. Moreover, if you don't use a proper API key Reddit will blacklist you *very* quickly, so I ended up doing the reasonable thing by spoofing my user agent as Chrome on Windows.

***Example output:***
```
TITLE: [GPU] MSI Airboost Rx Vega 56 $474.99 ($649 - 150 IR - 25 MIR)
LINK: https://www.reddit.com/r/buildapcsales/comments/8m7zdk/gpu_msi_airboost_rx_vega_56_47499_649_150_ir_25/

TITLE: [Graphics Card] EVGA GeForce GTX 1080 Ti SC2 ELITE GAMING - $749.99 ($909.99 - $160 + free shipping)
LINK: https://www.reddit.com/r/buildapcsales/comments/8m7muo/graphics_card_evga_geforce_gtx_1080_ti_sc2_elite/

TITLE: [GPU] PNY 1080 OC - $460($619 - Walmart Price Match -20% coupon)
LINK: https://www.reddit.com/r/buildapcsales/comments/8m7clk/gpu_pny_1080_oc_460619_walmart_price_match_20/

TITLE: [GPU] Powercolor Red Devil Vega 64 w/ Far Cry 5 - $599.99 ($619.99 - $20 MIR)
LINK: https://www.reddit.com/r/buildapcsales/comments/8m7172/gpu_powercolor_red_devil_vega_64_w_far_cry_5/
```
## Filters
I added Regex filters so it would only display posts that contain a specific pattern; in my case, `GPU`. The caveat of this was that it applied the filter to *all* of the subreddits. If you wanted to just show GPU posts from [r/buildapcsales][r_buildapcsales] and still get all posts from [r/hardwareswap][r_hardwareswap], you were pretty much out of luck.

## Split
It's worth making a distinction between the [first versions][first_versions] of this program and what it is now. It transitioned from a simple stand-alone Python script anyone could run to a Reddit bot anyone [could subscribe to][bot_frontend]*.

**Currently the bot only accepts whitelisted names since it's vulnerable to SQL injection...*

## Reddit Bot
Since I wanted push notifications on mobile, I needed to integrate with an app I already had on my phone. The logical choice presented itself almost immediately as Reddit, since I'm sending Reddit posts already.

I created [u/sub_notif_bot][u_sub_notif_bot] then generated my API key. 

Now that I was going to be using the Reddit API already, I might as well start using it to get posts as well.
## Multiple Users
Up until this point I had a single file `config.toml` that could be edited to change which subreddits to get posts from. This isn't ideal because

1. It doesn't scale to multiple users
2. It's not user friendly

The solution: a database. I hadn't ever messed with SQL or databases in general very much before so it was definitely an interesting challenge for me. 
I ended up with one table that had 3 columns. I figure their purpose should be pretty self explanatory.
```
sqlite> SELECT * FROM users;
username    subreddits     filters   
----------  -------------  ----------
kil0meters  buildapcsales  GPU
```

Now I basically just loop through the old program for each user then send them updates, but this is pretty inefficient.

## Frontend

I built a super basic [frontend][frontend] with Bootstrap and plain JS, using the fact that you can link to private messages in the format
```
https://www.reddit.com/message/compose/?to=<account>&subject=<subject>&message=<message>
```
This definitely beats editing a TOML/JSON file by hand!

## Scalability (And Better Filters)

As mentioned before, what I was currently doing was rather inefficient. For example, if two users are both subscribed to [r/buildapcsales][r_buildapcsales], it would request data from that subreddit twice.

Solving this problem is rather simple in theory; instead of iterating over a list of users, I iterate over a list of *subreddits*. After I have a list of posts sorted by subreddit, I iterate over the list of users, then just apply each users filtes and send them their posts.

This architectural change also allows us to solve the problem I had with filters before by editing our database a bit. I change the table `users` to only contain 2 columns; one per users, and one for the subreddits they have. In addition, I add a second table `filters` which has one column per subreddit per user, and an additional column with the username.

```
sqlite> SELECT * FROM filters;
username    subreddit      filter   
----------  -------------  ----------
kil0meters  buildapcsales  GPU
kil0meters  hardwareswap  
```
I can use basic SQL queries to get what I need e.g. 
```
SELECT subreddits FROM users WHERE username="USERNAME";
SELECT filter FROM filters WHERE username="USERNAME" AND subreddit="SUBREDDIT";
```
I've since realized I could do this all in one table by simply doing 
```
SELECT subreddit FROM filters where username="USERNAME";
```
Now I apply the filters per subreddit per user which means I can get shown only the posts I want, and without getting rid of things I don't want!

## Almost Done

Now, what would be a project without bugs?

Most of the bugs were rather tame. For example, where Reddit would blacklist my user. However, there was one that wasn't really *complicated per se*, but I had a hard time tracking down. 

The bug:

{% highlight python %}
+ most_recent_time, posts_by_subreddit = get_new_posts(posts_by_subreddit, most_recent_time)
  for user in user_list:
-   most_recent_time, posts_by_subreddit = get_new_posts(posts_by_subreddit, most_recent_time)
    posts_by_subreddit = apply_filters(posts_by_subreddit, user[1], user[0], db)
    try:
      posts_by_subreddit_by_users[user[0]] += posts_by_subreddit
{% endhighlight %}

Yeah, *that's it*. What was happening was that the bot was completely functional for me, but my friend got no posts. 
This is a result of how the bot determines which posts are actually new. The function `get_new_posts` would return only the posts that cam after `most_recent_time`, and set `most_recent_time` to the time of the most recent post. The bug here is that it does this *per user*, so by the time it gets to the second user, no posts will be new.

# The Future

Well, the bot is pretty much done, but now I have to think about it's underlying use cases and how to improve upon them.
The main use case of this bot is to get updates on "deal" subreddits, but this still requires you to wait for someone to actually post something. So, perhaps I should make a bot that cuts out the human entirely and browses sites directly looking for good deals...

[r_buildapcsales]: https://reddit.com/r/buildapcsales
[r_hardwareswap]: https://reddit.com/r/hardwareswap
[beginning]: https://github.com/kil0meters/subreddit_notifications/tree/4a21b04810325c6e87650d084daf65741467bf1e
[GRedditNotifier]: https://github.com/Jonatino/GRedditNotifier
[first_versions]: https://github.com/kil0meters/subreddit_notifications/tree/0fa1e252dfba6b1bc0326b858f4b7d05b0bb3b83
[bot_frontend]: https://kil0meters.github.io/sub_notif_bot
[u_sub_notif_bot]: https://reddit.com/u/sub_notif_bot
[frontend]: https://github.com/kil0meters/sub_notif_bot