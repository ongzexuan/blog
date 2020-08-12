---
title: "Thoughts on a Design Pattern for Facebook Chatbots"
date: 2018-07-18T00:49:17-04:00
author: Ze Xuan Ong
feature_image: false
draft: false
summary: "A compilation of post-mortem thoughts on building my first Facebook chat bot that is more than just a 'Hello World', mostly motivated that almost everyone seems to be just building a thing that works without much thought about the context."
---

A compilation of post-mortem thoughts on building my own Facebook Bot that is more than just a 'Hello World'.

## Design Patterns?

When I was starting out learning how to build a chatbot, it was nice to find that there was a good number of sites appearing to teach how to build the 'Hello World' equivalent of a chatbot. What I found particularly frustrating however, was that most if not all of these resources focus pretty much only on getting you through the door, and contain nearly nothing about what happens afterwards. This is particularly annoying when you have just found bots to be quite fun and want to think about scaling the bot (in terms of amount of traffic that can be handled concurrently) or integrating other services. After a few attempts at trying to make a robust enough base on Flask for a bot, I went with a relatively popular framework called Botkit. Even then, it doesn't seem as if there is much content written about design patterns for bots, Facebook or not. Scripting messages by hand within your code just seems to be very un-DRY and 'unclean'.

I have summarized some of my thoughts and experiences in this post and hope it will be helpful to anyone who wants to build a bot with some practical relevance beyond just a simple 'Hello World'. As I only built a bot for Facebook Messenger, there may be various platform specific features that may or may not matter to you. I will focus mainly on Facebook Messenger since I feel the content available out there pertaining to Facebook Messenger tends to be a bit sparse. Also, the use cases for say, Slack bots vs Facebook bots tend to be quite different, so take what you need out of this.

## Parts of a DIY Bot

More likely than not, you're designing a bot with a specific communication platform/channel in mind. This channel is also likely to be either Slack or Facebook Messenger. 

At this point, the slightly ambitious might decide to go it alone without any sort of framework. If you decide to build your own framework to send and receive messages, as I did for some time, then you have to consider constructing a number of relatively independent components, some of which affect the user experience significantly more than others:

* Server for sending and receiving messages from the relevant endpoints
* Filter for routing different types of messages to different handlers (e.g. buttons, text messages, delivery receipts etc. in Messenger)
* Logic and dialog modules for deciding the appropriate response
* Template generator to craft the response in a platform specific format

You may also want to have these things:

* Saving state and conversation context (i.e. things that user has told you)
* A means of differentiating concurrent users
* Database storage features
* Natural Language Processing (NLP) capabilities for free form responses
* Some sort of output randomization to make the bot less 'bot-ish'

The straw(s) that broke the donkey's back (I decided to revisit existing frameworks after this):

* A message queuing system to ensure that users get messages in the correct order and do not get spammed by your bot because of delays in messaging in parts of the system beyond your control
* A message delivery receipt system to check if messages have been delivered/received, and retry accordingly

If you're writing a bot for your own private use, then perhaps these actually pretty important features may not matter so much to you. After all, it's your own fault that you just received 20 messages in a row, and you learn to live with it.

Thankfully, frameworks exist, and while there plenty of room for improvement, they *suffice* for now.

## Separation of Bot and Conversation

Attempting to separate the 'content' from the actual 'bot' can be relatively tricky. Unlike web pages or apps where the content can be served in a relatively straightforward manner from a CMS, chatbots have 'content' that is more than just text. This content will include the logic of the dialog flow, and not to mention the input collection and corresponding data validation and storage processes. Hence, it is not quite possible to use a traditional CMS to serve the content. 

Does there exist some sort of CMS for editing chatbot dialog flows? If you discount the free services that build you a quick but simple chatbot, I seemed to only be able to find Botkit Studio in the market, and it's relatively limited (for reasons I'll explain later). This was also a very helpful push factor for adopting Botkit as the framework of choice, not because it was great, but because there wasn't any better alternative.

What sort of separation between the bot and conversation do we want exactly? Perhaps we can find the answer by looking at the outcomes that we want instead. Is it technically possible for say, another person or department to 'decide' on the dialog flows, while you implement the rest of the service? Could we perhaps agree upon a list of states and fields to be collected/validated, and then both sides can be off on their merry way, one building the hooks/validators/connectors, and the other building the relatively organic conversations?

I'd like to get there eventually, but unfortunately I think that is a difficult stage to reach now. In particular, a big issue is that it's pretty difficult for states to be completely decided beforehand. Perhaps after constructing the conversation, it becomes apparent that some sections are too lengthy, or a more timely transition message is warranted between questions. No conversation designer would commit in advance to X number of dialog bubbles. That's great for implementation, but no real natural conversation takes placed with a pre-defined number of exchanges.

### Blocks, Flows and Objectives

Assuming that you're building a task-oriented bot, as most people are, there generally is a predefined list of 'use cases' for the bot, just like in a use case diagram (seriously?)

{{< figure src="/design-pattern-facebook-chatbot/usecase.png" caption="(Image credit: Lucid Charts)" >}}

(Image credit: Lucid Charts)

One good way to organize the mess that is fluid conversation (at least, as far as a bot is concerned) is to segment the 'universe of dialog' into funnels/flows leading to certain 'use cases' or end goals. Of course, in real conversation you are likely to jump in and out of flows, but let's not worry about that for now (or maybe never! *evil laugh*). Then, since each flow can have a well-defined start and end state, it then just becomes a matter of connecting the dots from the start to the end.

For example, you might be building a food recommendation bot. One such conversation flow might be for the user to get recommendations based on current location. We can visualise the start being perhaps triggered by the user clicking a button or asking a question, and the end will likely be a list of locations followed by a thank you message. Now it becomes easy to figure out how to build the intermediate dialog bubbles, along with the information fields that need to be provided by the user. Another flow in the same bot might be to proactively push food near the user at certain times of the day.

In some sense, these 'flows' are somewhat similar to 'features'. It's much easier to agree upon the 'what' we want to build before we talk about the 'how'.

## Frameworks

I had initially used Botkit to build a simple proof of concept. This was before I had any real experience with Node.js and Javascript. The main motivation for me then was really the Botkit Studio that came with it, which allowed 'scripts' to be authored relatively independently of your bot.

I say 'relatively', because even for a supposedly simple task such as collecting user input, and then responding with that user input, this will require some work i.e. changes on the bot's end. Botkit provides for some simple message routing rules based on Regex and pattern matching, but not much more than that. To avoid strange conflicts with input collection and validation between the 'scripting' end and the 'backend', I decided to do all data validation on the bot itself, and leave the scripting engine to doing just scripts.

Using a framework helps to alleviate a lot of the initial time and energy investment in getting your bot to a relatively robust state. That being said, it may also come with a lot of superfluous tools that you won't need to use ever. Depends on your use case.

## Facebook Messaging

Fortunately or unfortunately, messaging platforms tends to provide a variety of ways for users to send messages. User replies to your bot via Facebook Messenger generally take one of three forms:

* Text messages
* Quick replies
* Postback buttons

I find it meaningful to distinguish between messages of the different types when handling messages. But more on that below.

### Text Messages
Text messages are usually plain text. However if you're using Messenger, be aware that users can also send strange attachments along with their messages. Text messages can be handled either through a very coarse method of pattern matching, or scanned for intent using some NLP engine. 

### Quick Replies
Facebook has this feature called quick replies, which to me seem like autocorrect suggestions. Although all quick reply 'buttons' are required to have a payload, I prefer to treat quick replies as simply text. i.e. I believe that quick replies are merely an easier way to input what otherwise would be free form text anyway. This is useful for generating suggestions to the user for a free response question, say for a food or music genre recommendation.

### Postback Buttons
A postback button is defined as such because clicking on it triggers a postback event, which Messenger will route to your webhook. A key defining feature of the postback buttons is their persistence in the message history. Unlike quick replies that disappear as long as the parent message is not the last message, postback buttons can stay long after their intended 'use case', perhaps even after the relevant trigger on the bot has been deprecated.

The implications that this has on the design of the bot and its conversations is:
* Users can click 'Yes' and 'No' multiple times in any permutation. If this is for a context sensitive request, then the corresponding result should be designed in an idempotent fashion. Otherwise, a more 'graceful' option would be to maintain a state to indicate if the user has previously responded to this question, and ask for a re-confirmation if the user clicks the buttons more than once
* Postback button triggers should be backwards compatible, or at the very least, there should be a fallback catch-all message response for all unknown postback button triggers.

### Do I need a button or a quick reply?

Suppose you have a conversation where you need a Yes/No response from the user. Would it be better to use a postback button or quick reply? The answer is that it depends on:

* Do you want to respond to text messages when the bot is in a state that is expecting a button press? Does it ignore, or send a default message, or get angry at the user?
* Do you want to handle the edge cases where your very exhaustive and comprehensive list of possible 'Yes' and 'No' replies fails to match 'k'?
* Do you care that your bot may seem more like an enlightened telephone menu rather than what the general media thinks of AI, blockchain, and machine learning?

If you answered Yes to any of the above, then perhaps a quick reply might be more suitable!

### Going back

I'm of the belief that a user should always have some way of exiting the current context. For mobile apps, if something goes horribly wrong, or the app just refuses to respond, one can always close the app on the task bar and restart it. Otherwise, there is always a 'back' button to return to the previous state. For bots, it is not immediately obvious what the course of action should be, particularly in cases where the bot not respond, responds inappropriately, or just doesn't provide the correct options for a user to respond to. There is also no native 'back' functionality provided, simply because there is no 'going back' in a conversation. Hence, it is necessary to provide a means to 'restart' the conversation or 'reboot' the bot. 

The first solution to this problem is to add a bunch of menu options in the persistent menu provided by Messenger. If you bot is a task-driven bot as opposed to an entertaining but objectiveless chatterbot, this would also make functional sense as it offers users a way to kick start certain processes to achieve some end.

The second solution is to send all input through a pattern matching process or NLP engine and identify if the user's intent is to get help, to return to the previous state, or to simply end the conversation. This means that these intents must be prioritized over the current conversation context. An obvious but common example would be:

```
Bot: Can I get your name please?
User: Bobb
Bot: Ok Bobb, can I get your current age?
User: Hold on I spelt that wrong, I mean Bob
Bot: I'm sorry I don't think that is a number! Can I get your age please?
User: ...
```

If your bot had any semblance of real intelligence to begin with, by the end of this conversation there will be no illusions about the non-intelligence of this entire ordeal.

## Channel Limitations

There are various limitations imposed by the channels through which you communicate with your user. You will probably find these if you read through the documentations a sufficient number of times, but here's a quick summary of what I think are the most significant limitations for Facebook Messenger.

* **Button text is limited to 20 characters**: Going beyond that limit will cause Messenger to render ellipses, which just doesn't really reflect nicely on your UI
* **Maximum of 3 buttons per message**: You only get to attach 3 buttons per message bubble. This is obviously quite limiting if you really need more than 3 options. An unsatisfactory workaround is to simply append more messages with more buttons. A better but more complex workaround is to use quick replies with some pattern matching or NLP intent matching to categorize the user response
* **No hyperlinks in text**: You can't put hyperlinks in text. You have to use a button (buttons are expensive clearly)

Another limitation that may not be obvious at the start, but the chat window of Messenger is very small on the desktop, unless you are using messenger.com and not the chat window on Facebook. This limits how much text you can reasonably send in a short burst. The workaround to that is to space out your messages with delays, or to use the typing indicator to let users anticipate an incoming message. The problem this then exposes is that you would have to handle the case where users send new messages (interuption) while your bot has not completed its last volley of messages.

Before you start planning on what your dialog flows are going to look like and how your responses are going to be crafted, take a good look at the channel through which your responses will be routed, and picture how much content is reasonably digestible in a defined span of time.

## Conversations in Botkit

In constructing conversations, it is most likely that you would want some concept of a context and state. The context defines what the topic is, it may simply be referring to a specific conversation flow that was designed. The state defines which point in the conversation the user is currently in. This is important for determining the next response, or deciding if input needs to be collected and validated in the next response.

As mentioned, I wound up using Botkit and Botkit Studio for my project. There are some pretty nifty features already built in, including a message queuing system, a conversation context, and the ability to send and receive messages from various channels (Messenger included of course). This isn't a post about Botkit per se, but some of the features I felt were necessary have by and large been implemented in Botkit, though not with as much flexibility as I would have preferred.

Botkit has the concept of a 'conversation'. This is akin to a context in which messages are somehow related and routed to one another, and where there can be saved variables e.g. user's name. Within the conversation, you have multiple of something you call a 'thread'. I still find this slightly misleading. I was expecting a thread to contain a list of back and forth responses between user and bot, but the usual structure of a thread typically comprises of a message or two from the bot, followed either by waiting for user input (and then routing to another thread), or redirection to another thread, or terminating the conversation. Perhaps it may make more sense if you simply think about 'threads' as 'states', and the scripts as 'conversations'. You can define these conversations on Botkit Studio, or within the bot itself. Botkit Studio has a free trial, thereafter you pay a small subscription fee for a fixed number of API calls. If you've used it yourself, the interface looks slightly dodgy, or maybe its a new paradigm of design that I have yet to appreciate, but I must say it is probably the best thing that you can really get out there right now.

### But postbacks

As mentioned previously, postback buttons on Messenger trigger a different event than ordinary messages. This is good for allowing differentiation of response priority by event type, but it turns out that this is bad for trying to maintain a conversation context, since if we prioritize a postback message over a text message, then by default it is always outside every conversation context. That means each time you click a button, you have created/entered a new context, and god knows what has happened to the last one. If you choose to treat postback buttons as text instead, then you would have to account for all the postback buttons that matter in each conversation, and that would be a nightmare to script, not to mention code. It also does not seem likely that we will be able to segment postback buttons into different 'groups' anytime soon, whether from Facebook's end or Botkit's end.

The lesser of two evils than perhaps is to go with the former and enter a new conversation with each button press. I have also very purposefully omitted to look at the efficiency of terminating all existing conversations each time a button is pressed.

## Designing the dialog flow

Having previously felt quite let down by the fact that no bot scripting CMS really exists out there in the market, I thought perhaps at the very least some sort of dialog design tool might exist as a common platform for all stakeholders to visualize the possible conversations. I was looking for something perhaps like InVision or Figma. As it turns out, perhaps less surprisingly this time, such a mythical tool doesn't quite exist.

Going back to basics then, in theory one could use PowerPoint (it's Turing complete anyway right) or its equivalents to sketch this dialog flow. The main gripes I have would be:

1. **How scalable is this canvas**: dialogs may scale/grow out of control very fast if you don't define the project requirements concretely!
2. **How can I account for randomised responses**: Botkit provides for a simple way to create randomized responses to reduce repitition and provide some semblance of intelligence. But I wouldn't want to duplicate ten space occupying bubbles just to show that I have multiple responses
3. **How can I account for all the edge cases**: A dialog flow tends to assume the best about the user responses. However, I need a means to indicate the bot's responses to unexpected user input. This includes providing invalid responses to questions, timeouts and drop-offs, or entering text with certain intents.

I have yet to find a solution that relatively satisfactorily satisfies half of these. Currently, I use Google Drawings to sketch the dialog flows, largely because it satisfies requirement 1, and is relatively straightforward to use.

There is some market for a solution! 

## NLP Engines?

I have largely omitted to talk about NLP engines. Actually I've tried a good number of them, specifically Wit.AI, DialogFlow (API.AI), and IBM Watson. I would say I enjoyed DialogFlow the most, though they lost one of my dialog agents (i.e. bots) which still has yet to find its way back to me.

All the platforms provide some sort of intent and entity detection, which I believe is the bare minimum. One thing that I wish was provided would be some sort of text disambiguation/normalization (think bucket sort). The use case of this would be for collecting free text input from a user for what really is a list of options. For example you might be asking a user for what food they like to eat, but really what you're interested in is the type of cuisine and not so much the specific dish, so you would want to sort free input into some bucket of cusines. One could use an entity to do this, but most NLP engines won't return you an entity without some sort of associated intent (no intent no entity). This means you then have to construct an intent, but then you'd need to update both the intent and the entity when you create new values. This process also adds noise to the intent determination process, which is annoying when you know for certain that you only require normalization when you're in a certain state but not others.

Wit.AI is pretty straightforward and no frills. It does what it needs to do, though I have a slight gripe with not being able to enter a large amount of data at a time, and making edits is terribly difficult (because it's difficult to access the saved entries).

IBM Watson provided a rather decent dialog scripting tool. The downside to that was that it was not obvious how I could insert custom validators for input and extract input to a database in the middle of the flows. For a Q&A bot it would work fine.

DialogFlow tries to provide a good number of additional features beyond the usual, including fulfilment options and even a context. Right now however, given that I do not know the full extent to which I can leverage on the NLP engine, I'd prefer to keep track of my context and state myself. Furthermore, it's not clear exactly how these new features are meant to be used without example design patterns (which is one of the motivating factors for this post). I intend to explore this in greater detail in my next project.

## Looking forward

Currently, I see the chatbot merely as a simpler interface to retrieve information and to perform actions. This is relevant because there may be tons of information pertaining to your site that is hidden or nested in some corner that is simply just too much of a hassle to click, or even open as a URL. I'll probably find that information if I dig hard enough, but why should I? The chatbot thus provides a lower barrier to entry interface for users to get what they want immediately without leaving the comfort of their messaging platform, or opening a URL that loads a lot of extraneous information. Plus, the UI comes for free.

This view also means that, if your chatbot does not simplify the information retrieval process, and your goal isn't to entertain people with meaningless conversation, then you need to relook at your dialog flows or end objective of the conversations that you are constructing. There are a number of articles already written on related topics.

### A task-oriented chatterbot?

I'm interested in trying to figure out if it is possible to meld the task-driven bots with chatterbots to create a less machinated question answering system. This is because most (Facebook) pages probably need some sort of FAQ system. Beyond a certain size of FAQ, it becomes less viable to have a laundry list of items. Instead, a search or chat solution might make more sense. Currently approaches to chat however, treat the chatbot as nothing more than another search box, which I feel greatly underutilizes the potential of the chat medium. Consider

```
User: Is the cafe busy around 2pm today?
Bot: Here's a chart (link) of our busy times (sourced from Google)
```

Or slightly better

```
User: Is the cafe busy around 2pm today?
Bot: We are busiest at 1pm on weekdays!
```

The most reasonable response can usually be elucidated from putting yourself in the shoes of your most conscientious customer service representative.

```
User: Is the cafe busy around 2pm today?
Bot: We're usually quite busy around that time on Wednesdays.
Bot: If you could come earlier around 1pm, we're a bit less busy then!
```

Where the task driven side of the bot reflects the use cases and pieces of information a user could possible derive from the service, the chatterbot side would represent a culture and personality. I think that some combination of a ChatScript engine with the Botkit framework might suffice for this role, though that would demand more from the dialog designer.

## In Sum

Chatbots are kind of cool and somewhat underutilised in my opinion. It's difficult to scale, and requires quite a ton of upfront investment in terms of development time and energy to make something that doesn't feel like it was hacked together in a day.
