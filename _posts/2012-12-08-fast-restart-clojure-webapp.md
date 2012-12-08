---
author: feng
date: '2012-12-09 22-21-00'
layout: post
status: publish
title: I make my Clojure web app restarts faster
categories: ['rssminer', 'clojure']
---

### Rssminer restart slow, which causes seconds service outage

[**Rssminer**, http://rssminer.net](http://rssminer.net) is a web app write in Clojure. The app restart slow, which make me headache. The deployment process:

1. [ahead-of-time compile](http://clojure.org/compilation) all Clojure code into JVM bytecode
2. Pack them along with all dependencies into a single jar: rssminer-standalone.jar using `lein uberjar`
3. rsync rssminer-standalone.jar to production server
4. Kill previous Rssminer process
5. Start a new one by running

{% highlight sh %}
nohup java -jar rssminer-standalone.jar # listens on port 8100, reverse proxy by Nginx
{% endhighlight %}


But there is a time window between program killed and restarted, in about 10 seconds or so

**This is service outage, users will notice when are using it: Nginx returns 502 during the time**

Which is bad, must be solved if I want users trust the service.

Rolling release is not possible since the app get only one production server.

### Solution: kill only after app started
Here is one working solution I found yesterday night, after get home from work:

After step 3 is done, do not kill process immediately. But run step 4.
In rssminer-standalone.jar, there is some logic after program get loaded and initialized, but before socket is bind to 8100, something like this:

{% highlight sh %}
lsof -t -sTCP:LISTEN  -i:8100 | xargs kill  # kills old instance
{% endhighlight %}

After old instance get killed, bind to 8100.

The logic is easy to write, especially in Clojure.
Here is the relevant code

{% highlight clojure %}

(loop [i 1]
  (let [pid (str/trim (:out (sh "lsof"
                                "-t" "-sTCP:LISTEN"
                                (str "-i:" (cfg :port)))))]
    (when-not (str/blank? pid)
      ;; old program is runing, kill it
      (info "kill pid" pid i "times, status" (:exit (sh "kill" pid)))))
  (let [r (try
            ;; bind socket, start web server
            (reset! server (run-server (app) {:port (cfg :port)
                                              :ip (cfg :bind-ip)
                                              :worker-name-prefix "w"
                                              :thread (cfg :worker)}))
            1 ;; return 1 means success
            (catch java.net.BindException e
              (if (> i 30)    ; wait about 4.5s
                (do
                  (info "giving up" e)
                  (throw e))
                (Thread/sleep 150))))]
    (when-not r
      (recur (inc i)))))

{% endhighlight %}

### Testing

Two shell running:
{% highlight sh %}
for in in {1..100}; do java -jar rssminer-standalone.jar; done
{% endhighlight %}

Rssminer starts, get killed, start again, killed again ....

I constantly refresh browser, No noticeable downtime.

### About Rssminer

[Rssminer](http://rssminer.net) is a RSS reader build by me using Clojure. You can try a [demo here](http://rssminer.net/demo)
