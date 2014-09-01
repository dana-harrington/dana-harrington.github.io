---
layout: post
title: "Proxying Web Services With a Monad"
description: ""
category: monads
tags: [monads, web services]
---
In this article I'll describe how a monad can be constructed to manage cookie handling in an intermediate layer that acts as a proxy for the end user to make calls through to a backend web service. The monad structure will take care of collecting cookies sent back by the web service, and handle collecting and chaining cookies through multiple back end calls.
