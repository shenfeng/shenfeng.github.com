---
author: feng
date: '2011-09-03 20-21-00'
layout: post
status: publish
title: Using TagSoup to extract text from HTML
categories: ['clojure']
---


These days, I am experimenting
[Apache Lucene](http://lucene.apache.org/java/docs/index.html). I need
a way to extract text from HTML source, feed it to Lucene. I first
come up with a solution by using regex and Clojure:

{% highlight clojure %}
(defn extract [html]
  (when html
    (str/replace html #"(?m)<[^<>]+>|\n" "")))
{% endhighlight %}

Most of the time, it works, and very fast. But it can't ignore
Javascript and CSS, which is needed. So I come up with another
solution, by using [enlive](https://github.com/cgrand/enlive ).

Here is the Clojure code.

{% highlight clojure %}
 (defn- emit-str [node]
   (cond (string? node) node
         (and (:tag node)
              (not= :script (:tag node))) (emit-str (:content node))
         (seq? node) (map emit-str node)
         :else ""))
 (defn extract-text [html]
   (when html
     (let [r (html/html-resource (java.io.StringReader. html))]
       (str/trim (apply str (flatten (emit-str r)))))))
{% endhighlight %}

It's works. javascript is ignored. But it's a little slow:
On my machine, extract a given html file, regex takes 0.21ms,
But extract-text takes 2.76ms.

Enlive is build on top of
 [TagSoup](http://home.ccil.org/~cowan/tagsoup/), which a SAX-compliant parser
written in Java that, instead of parsing well-formed or valid XML,
parses HTML as it is found in the wild: poor, nasty and brutish,
though quite often far from short.

By calling

{% highlight clojure %}
(html/html-resource (java.io.StringReader. html)
{% endhighlight %}

Enlive build a tree for the html, which is a little overkill for only
extract text. By directly using TagSoup, I can bypass this overhead.
Here is the Java code:

{% highlight java %}
public class Utils {
    public static String extractText(String html) throws IOException,
            SAXException {
        Parser p = new Parser();
        Handler h = new Handler();
        p.setContentHandler(h);
        p.parse(new InputSource(new StringReader(html)));
        return h.getText();
    }
}
class Handler extends DefaultHandler {
    private StringBuilder sb = new StringBuilder();
    private boolean keep = true;
    public void characters(char[] ch, int start, int length)
            throws SAXException {
        if (keep) {
            sb.append(ch, start, length);
        }
    }
    public String getText() {
        return sb.toString();
    }
    public void startElement(String uri, String localName, String qName,
            Attributes atts) throws SAXException {
        if (localName.equalsIgnoreCase("script")) {
            keep = false;
        }
    }
    public void endElement(String uri, String localName, String qName)
            throws SAXException {
        keep = true;
    }
}
{% endhighlight %}

After experiment, I find

{% highlight java %}
Parser p = new Parser();
{% endhighlight %}

takes a lot of CPU time.
By using
[ThreadLocal](http://download.oracle.com/javase/6/docs/api/java/lang/ThreadLocal.html)

{% highlight java %}
private static final ThreadLocal<Parser> parser = new ThreadLocal<Parser>() {
    protected Parser initialValue() {
        return new Parser();
    }
};
{% endhighlight %}

It takes 0.38ms to extract text from the same html file. I am happy
with the result.
