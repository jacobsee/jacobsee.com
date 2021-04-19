---
title: "Short Stuff: Let Me Paste Passwords!"
description: "Password pasting - we can do this the easy way or the hard way"
date: 2021-04-18
---

I have to be honest - I thought we were done with this phase of Internet "security" [a long time ago](https://www.wired.com/2015/07/websites-please-stop-blocking-password-managers-2015/). Don't we all use password managers these days? Does anybody actually know their password to any major website anymore? I don't. That's why I was so surprised to run across a website (_cough_ **Costco** _cough_) that was still disabling paste on password inputs... at least, on their registration page. &nbsp;&nbsp;<sup>yes it took me this long to make a Costco account...</sup>

![](/images/costco-disable-paste.png)

It was at this point I decided that instead of opening my password manager in a new window and just typing the very long generated password into this registration page twice, I would spend _significantly more_ time to fix the problem itself.

Obviously, we can't edit anything server-side... but for this, we don't really need to. The code executing this betrayal is just JavaScript running in our own browser. A-la _"We've traced the call. It's coming from inside the house."_ And we can do whatever we want inside of our own house.

One convenient way to inject our own JavaScript in the browser is to run a plugin such as [Tampermonkey](https://www.tampermonkey.net/) ([Firefox](https://addons.mozilla.org/en-US/firefox/addon/tampermonkey/), [Chrome](https://chrome.google.com/webstore/detail/tampermonkey/dhdgffkkebhmkfjojejmpbldmpobfkfo?hl=en)). Tampermonkey provides an environment for you to write your own scripts (or use public scripts published by others), and specify the URLs on which those scripts should activate and run.

Luckily for us - this is a very simple problem to solve with a script! As seen in the screenshot above, they simply attach an event handler to the `paste` event and then `return false`, effectively canceling the paste.

To fix pasting, we need to intercept the event before it reaches this handler and do something else. Click on the Tampermonkey icon on your taskbar, go to the dashboard, and create a new script with the following:

```javascript
// ==UserScript==
// @name         Allow Pasting
// @namespace    https://jacobsee.com
// @version      0.1
// @description  Allow pasting passwords on sites that try to disable it
// @author       Jacob See
// @match        https://www.costco.com/*
// @icon         https://www.google.com/s2/favicons?domain=costco.com
// @grant        none
// ==/UserScript==

(function() {
    var youllNeverGetMyPaste = function(e){
        e.stopImmediatePropagation();
        return true;
    };
    document.addEventListener('paste', youllNeverGetMyPaste, true);
})();
```

This script adds its own ["capturing"](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Building_blocks/Events#event_bubbling_and_capture) event listener for the `paste` event across the entire document, with a handler that _prevents propagation of that event to other handlers_, and "accepts" the paste. At this point, you can save the script and exit Tampermonkey.

Open a new tab and navigate to Costco, and the Tampermonkey icon should illuminate, indicating that a script is active! The original evil `paste` event handler still exists on the page, but it doesn't matter because the handler defined in our script takes care of it first!

Now we can paste our massive, inconvenient passwords to our heart's content. 😎
