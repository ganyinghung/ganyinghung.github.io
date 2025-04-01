---
layout: single
title:  "Multi-selectable FlatList"
date:   2021-05-06 10:00:00 +0100
categories: dev
---

Hi, this is my first post on hashnode. In this post I want to talk about how I implement a **multi-selectable `FlatList`**. 

<!--more-->
(This post was originally posted in my [hashnode](https://notes.yhg.io/multi-selectable-flatlist) blog.)

It is about [React Native](https://reactnative.dev), a way to write cross-platform mobile app using the [React JS](https://reactjs.org) UI library. I assume you have knowledge on both of them. If not, see [here](https://reactjs.org/tutorial/tutorial.html) and [here](https://reactnative.dev/docs/getting-started) for a great introduction.

### Let's Begin

First let's look at `FlatList`. `FlatList` is [a component in React Native](https://reactnative.dev/docs/using-a-listview)  which provides an easy, performant way to implement a scrollable list, something you've seen a lot in mobile apps. A `FlatList` requires, essentially, only two props:

1. `data`: an array of items, and
2. `renderItem`: a function that takes an item and returns a React element, ie. a "renderer".

Let's take a look at this simple example:

<div data-snack-id="@yhgan/simple-flatlist" data-snack-platform="web" data-snack-preview="true" data-snack-theme="light" style="overflow:hidden;background:#fbfcfd;border:1px solid var(--color-border);border-radius:4px;height:505px;width:100%"></div>
<script async src="https://snack.expo.dev/embed.js"></script>

Easy, right? The good thing about `FlatList` is that it comes with a lot of features out of the box, such as scrolling, multiple columns, infinite scrolling, and even pull to refresh. However, one thing it doesn't support is selection. So we would have to implement our owns.

In most cases, pressing an item goes to a "detailed view" of that item, and long-pressing it to select it. We will adapt this UI norm here. To support pressing/long-pressing, we use a React Native component called [`TouchableWithoutFeedback`](https://reactnative.dev/docs/handling-touches#touchables). We will use it to wrap our item view, and use its `onLongPress` prop to handle the long-pressing. We will also add a state variable to store the selected item's ID:

<div data-snack-id="@yhgan/simple-selectable-flatlist-not-working" data-snack-platform="web" data-snack-preview="true" data-snack-theme="light" style="overflow:hidden;background:#fbfcfd;border:1px solid var(--color-border);border-radius:4px;height:505px;width:100%"></div>


It looks good, but it doesn't work! Why? Because we need to tell our `FlatList` that something has changed and a re-render is needed. We use its `extraData` prop to do so, by setting it to our state variable:

<div data-snack-id="@yhgan/simple-selectable-flatlist-working" data-snack-platform="web" data-snack-preview="true" data-snack-theme="light" style="overflow:hidden;background:#fbfcfd;border:1px solid var(--color-border);border-radius:4px;height:505px;width:100%"></div>


Horray! It works! We have our selectable `FlatList`!

### Multiple Selection

Now we move further on, to implement a multi-selectable `FlatList`.

It basically has the same idea, but instead of a single state value, we use an *array*. Moreover, we cannot simply just set/resetting it, because we also need to handle the case when we want to deselect an item. Therefore, the long-press handler will now be a "toggle" function: if the item is not in our array, we push it in; otherwise, we splice it out.

<div data-snack-id="@yhgan/simple-multi-selectable-flatlist" data-snack-platform="web" data-snack-preview="true" data-snack-theme="light" style="overflow:hidden;background:#fbfcfd;border:1px solid var(--color-border);border-radius:4px;height:505px;width:100%"></div>


It works pretty well, isn't it? With just a little effort, we have our multi-selectable `FlatList`.

### Scalability

Finally, we want to see how scalable it is. Instead of 3 items, we increased it to, say, 50000 items:

<div data-snack-id="@yhgan/simple-multi-selectable-flatlist-50000-items" data-snack-platform="web" data-snack-preview="true" data-snack-theme="light" style="overflow:hidden;background:#fbfcfd;border:1px solid var(--color-border);border-radius:4px;height:505px;width:100%"></div>


(If you have the [Expo app](https://expo.io), feel free to test it on your device too.)

Turns out it still runs pretty smoothly, thanks to the excellent implementation of `FlatList`. Of course it also depends on how complicated your item component is. In fact, they have a [whole section](https://reactnative.dev/docs/optimizing-flatlist-configuration) on how you can optimize  `FlatList`. But in our case, it is good to know our way of handling multiple selection won't hurt its performance.

Thank you so much for reading! I hope your find my very first post on hashnode useful.













