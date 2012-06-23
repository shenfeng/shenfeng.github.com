---
author: feng
date: '2012-06-22 22-03-00'
layout: post
status: publish
title: About Javascript dispatch event and delegate event
categories:
- javascript
---

{% highlight js %}

(function () {
  var all_events = {};          // events data, private to this file
  function bind_event (ele, event, handler) {
    if(ele.addEventListener) {
      ele.addEventListener(event, handler, false);
    } else {                    // ie
      ele.attachEvent('on' + event, handler);
    }
  }
  // a Simple impl of dispatch event.
  function dispatch_event (ele, selector, eventName, handler) {
    var handlers = all_events[ele] || {};
    if(handlers[eventName]) {
      handlers[eventName].push(handler); // add more
    } else {
      handlers[eventName] = [handler]; // register
      var one_handler = function (e) {
        // find registerd handlers
        var handlers = (all_events[ele] || {})[eventName];
        if(!handlers) { return; }
        var target = e.target;
        // IE6, IE7 does not support querySelectorAll
        var nodes = ele.querySelectorAll(selector);
        for(var i = 0; i < nodes.length; i++) {
          // interested event hanpend on interested DOM node
          if(nodes[i] === target) {
            for(var j = 0; j < handlers.length; j++) { // call one by one
              // properly bind `this`
              var ret = handlers[j].apply(target, [e]);
              if(ret === false) {
                e.preventDefault(); e.stopPropagation();
              }
            }
          }
        }
      };
      bind_event(ele, eventName, one_handler);
    }
    all_events[ele] = handlers;
  }
  var EVENTSPLITTER = /^(\S+)\s*(.*)$/;
  // extracted from Backbone.js, brilliant
  function delegate_events (ele, events) {
    for (var key in events) {
      var method = events[key],
          match = key.match(EVENTSPLITTER),
          eventName = match[1],
          selector = match[2];
      if (selector === '') {
        bind_event(ele, eventName, method);
      } else {
        dispatch_event(ele, selector, eventName, method);
      }
    }
  }
  function click_handler1 () { console.log('click', this, e); }
  function click_handler2 () { console.log('click2', this, e); }
  // I like the syntax.
  // And it's very flexable, modify the DOM as you want,
  // event handler will work just the way you want it
  delegate_events(document, {
    'click li': click_handler1,
    'click ul li': click_handler2
  });
})();

{% endhighlight %}
