---
layout: post
title: Implementing Loaders (part 3)
date: 2012-08-21
permalink: /2012/08/implementing-loaders.html
comments: true
---

<p>This post introduces the <code>Loader&lt;D&gt;</code> class as well as custom Loader implementations. This is the third of a series of posts I will be writing on Loaders and the LoaderManager:</p>

<ul>
<li><b>Part 1:</b> <a href="http://www.androiddesignpatterns.com/2012/07/loaders-and-loadermanager-background.html">Life Before Loaders</a></li>
<li><b>Part 2:</b> <a href="http://www.androiddesignpatterns.com/2012/07/understanding-loadermanager.html">Understanding the LoaderManager</a></li>
<li><b>Part 3:</b> <a href="http://www.androiddesignpatterns.com/2012/08/implementing-loaders.html">Implementing Loaders</a></li>
<li><b>Part 4:</b> <a href="http://www.androiddesignpatterns.com/2012/09/tutorial-loader-loadermanager.html">Tutorial: AppListLoader</a></li>
</ul>

<p>First things first, if you haven’t read my previous two posts, I suggest you do so before continuing further. Here is a very brief summary of what this blog has covered so far. <a href="http://www.androiddesignpatterns.com/2012/07/loaders-and-loadermanager-background.html">“Life Before Loaders” (part 1)</a> described the flaws of the pre-Honeycomb 3.0 API and its tendency to perform lengthy queries on the main UI thread. These UI-unfriendly APIs resulted in unresponsive applications and were the primary motivation for introducing the Loader and the LoaderManager in Android 3.0. <a href="http://www.androiddesignpatterns.com/2012/07/understanding-loadermanager.html">“Understanding the LoaderManager” (part 2)</a> introduced the LoaderManager class and its role in delivering asynchronously loaded data to the client. The LoaderManager manages its Loaders across the Activity/Fragment lifecycle and can retain loaded data across trivial configuration changes.</p>

<h4>Loader Basics</h4>

<p>Loaders are responsible for performing queries on a separate thread, monitoring the data source for changes, and delivering new results to a registered listener (usually the LoaderManager) when changes are detected. These characteristics make Loaders a powerful addition to the Android SDK for several reasons:</p>

<ol>

<li value="1"><p><b>They encapsulate the actual loading of data.</b> The Activity/Fragment no longer needs to know how to load data. Instead, the Activity/Fragment delegates the task to the Loader, which carries out the request behind the scenes and has its results delivered back to the Activity/Fragment.</p></li>

<li value="2"><p><b>They abstract out the idea of threads from the client.</b> The Activity/Fragment does not need to worry about offloading queries to a separate thread, as the Loader will do this automatically. This reduces code complexity and eliminates potential thread-related bugs.</p></li>

<li value="3"><p><b>They are entirely <i>event-driven</i>.</b> Loaders monitor the underlying data source and automatically perform new loads for up-to-date results when changes are detected. This makes working with Loaders an absolute pleasure, as the client can simply trust that the Loader will auto-update its data on its own. All the Activity/Fragment has to do is initialize the Loader and respond to any results that might be delivered. Everything in between is done by the Loader.</p></li>

</ol>

<p>Loaders are a somewhat advanced topic and may take some time getting used to. We begin by analyzing its four defining characteristics in the next section.</p>

<!--more-->

<h4>What Makes Up a Loader?</h4>

<p>There are four characteristics which ultimately determine a Loader’s behavior:</p>

<ol>

<li value="1"><p><b>A task to perform the asynchronous load.</b> To ensure that loads are done on a separate thread, subclasses should extend <code>AsyncTaskLoader&lt;D&gt;</code> as opposed to the <code>Loader&lt;D&gt;</code> class. <code>AsyncTaskLoader&lt;D&gt;</code> is an abstract Loader which provides an <code>AsyncTask</code> to do its work. When subclassed, implementing the asynchronous task is as simple as implementing the abstract <code>loadInBackground()</code> method, which is called on a worker thread to perform the data load.</p></li>

<li value="2"><p><b>A registered listener to receive the Loader’s results when it completes a load.<a href="#footnote1"><sup>1</sup></a></b> For each of its Loaders, the LoaderManager registers an <code>OnLoadCompleteListener&lt;D&gt;</code> which will forward the Loader’s delivered results to the client with a call to <code>onLoadFinished(Loader&lt;D&gt; loader, D result)</code>. Loaders should deliver results to these registered listeners with a call to <code>Loader#deliverResult(D result)</code>.</p></li>

<li value="3"><p><b>One of three<a href="#footnote2"><sup>2</sup></a> distinct states.</b> Any given Loader will either be in a <i>started</i>, <i>stopped</i>, or <i>reset</i> state:</p>

<p>(a) Loaders in a <i>started state</i> execute loads and may deliver their results to the listener at any time. Started Loaders should monitor for changes and perform new loads when changes are detected. Once started, the Loader will remain in a started state until it is either stopped or reset. This is the only state in which <code>onLoadFinished</code> will ever be called.</p>

<p>(b) Loaders in a <i>stopped state</i> continue to monitor for changes but should <b>not</b> deliver results to the client. From a stopped state, the Loader may either be started or reset.</p>

<p>(c) Loaders in a <i>reset state</i> should <b>not</b> execute new loads, should <b>not</b> deliver new results, and should <b>not</b> monitor for changes. When a loader enters a reset state, it should invalidate and free any data associated with it for garbage collection (likewise, the client should make sure they remove any references to this data, since it will no longer be available). More often than not, reset Loaders will never be called again; however, in some cases they may be started, so they should be able to start running properly again if necessary.</p>

</li>

<li value="4"><p><b>An observer to receive notifications when the data source has changed.</b> Loaders should implement an observer of some sort (i.e. a <code>ContentObserver</code>, a <code>BroadcastReceiver</code>, etc.) to monitor the underlying data source for changes. When a change is detected, the observer should call <code>Loader#onContentChanged()</code>, which will either (a) force a new load if the Loader is in a started state or, (b) raise a flag indicating that a change has been made so that if the Loader is ever started again, it will know that it should reload its data.</p>

</li>

</ol>

<p>By now you should have a basic understanding of how Loaders work. If not, I suggest you let it sink in for a bit and come back later to read through once more (reading the <a href="http://developer.android.com/reference/android/content/Loader.html">documentation</a> never hurts either!). That being said, let’s get our hands dirty with the actual code!</p>

<h4>Implementing the Loader</h4>

<p>As I stated earlier, there is a lot that you must keep in mind when implementing your own custom Loaders. Subclasses must implement <code>loadInBackground()</code> and should override <code>onStartLoading()</code>, <code>onStopLoading()</code>, <code>onReset()</code>, <code>onCanceled()</code>, and <code>deliverResult(D results)</code> to achieve a fully functioning Loader. Overriding these methods is very important as the LoaderManager will call them regularly depending on the state of the Activity/Fragment lifecycle. For example, when an Activity is first started, the Activity instructs the LoaderManager to start each of its Loaders in <code>Activity#onStart()</code>. If a Loader is not already started, the LoaderManager calls <code>startLoading()</code>, which puts the Loader in a started state and immediately calls the Loader’s <code>onStartLoading()</code> method. In other words, a lot of work that the LoaderManager does behind the scenes <b>relies on the Loader being correctly implemented</b>, so don’t take the task of implementing these methods lightly!</p>

<p>The code below serves as a template of what a Loader implementation typically looks like. The <code>SampleLoader</code> queries a list of <code>SampleItem</code> objects and delivers a <code>List&lt;SampleItem&gt;</code> to the client:</p>

<p>
<pre class="brush:java">
public class SampleLoader extends AsyncTaskLoader&lt;List&lt;SampleItem&gt;&gt; {

  // We hold a reference to the Loader’s data here.
  private List&lt;SampleItem&gt; mData;

  public SampleLoader(Context ctx) {
    // Loaders may be used across multiple Activitys (assuming they aren't
    // bound to the LoaderManager), so NEVER hold a reference to the context
    // directly. Doing so will cause you to leak an entire Activity's context.
    // The superclass constructor will store a reference to the Application
    // Context instead, and can be retrieved with a call to getContext().
    super(ctx);
  }

  /****************************************************/
  /** (1) A task that performs the asynchronous load **/
  /****************************************************/

  @Override
  public List&lt;SampleItem&gt; loadInBackground() {
    // This method is called on a background thread and should generate a
    // new set of data to be delivered back to the client.
    List&lt;SampleItem&gt; data = new ArrayList&lt;SampleItem&gt;();

    // TODO: Perform the query here and add the results to 'data'.

    return data;
  }

  /********************************************************/
  /** (2) Deliver the results to the registered listener **/
  /********************************************************/

  @Override
  public void deliverResult(List&lt;SampleItem&gt; data) {
    if (isReset()) {
      // The Loader has been reset; ignore the result and invalidate the data.
      releaseResources(data);
      return;
    }

    // Hold a reference to the old data so it doesn't get garbage collected.
    // We must protect it until the new data has been delivered.
    List&lt;SampleItem&gt; oldData = mData;
    mData = data;

    if (isStarted()) {
      // If the Loader is in a started state, deliver the results to the
      // client. The superclass method does this for us.
      super.deliverResult(data);
    }

    // Invalidate the old data as we don't need it any more.
    if (oldData != null && oldData != data) {
      releaseResources(oldData);
    }
  }

  /*********************************************************/
  /** (3) Implement the Loader’s state-dependent behavior **/
  /*********************************************************/

  @Override
  protected void onStartLoading() {
    if (mData != null) {
      // Deliver any previously loaded data immediately.
      deliverResult(mData);
    }

    // Begin monitoring the underlying data source.
    if (mObserver == null) {
      mObserver = new SampleObserver();
      // TODO: register the observer
    }

    if (takeContentChanged() || mData == null) {
      // When the observer detects a change, it should call onContentChanged()
      // on the Loader, which will cause the next call to takeContentChanged()
      // to return true. If this is ever the case (or if the current data is
      // null), we force a new load.
      forceLoad();
    }
  }

  @Override
  protected void onStopLoading() {
    // The Loader is in a stopped state, so we should attempt to cancel the 
    // current load (if there is one).
    cancelLoad();

    // Note that we leave the observer as is. Loaders in a stopped state
    // should still monitor the data source for changes so that the Loader
    // will know to force a new load if it is ever started again.
  }

  @Override
  protected void onReset() {
    // Ensure the loader has been stopped.
    onStopLoading();

    // At this point we can release the resources associated with 'mData'.
    if (mData != null) {
      releaseResources(mData);
      mData = null;
    }

    // The Loader is being reset, so we should stop monitoring for changes.
    if (mObserver != null) {
      // TODO: unregister the observer
      mObserver = null;
    }
  }

  @Override
  public void onCanceled(List&lt;SampleItem&gt; data) {
    // Attempt to cancel the current asynchronous load.
    super.onCanceled(data);

    // The load has been canceled, so we should release the resources
    // associated with 'data'.
    releaseResources(data);
  }

  private void releaseResources(List&lt;SampleItem&gt; data) {
    // For a simple List, there is nothing to do. For something like a Cursor, we 
    // would close it in this method. All resources associated with the Loader
    // should be released here.
  }

  /*********************************************************************/
  /** (4) Observer which receives notifications when the data changes **/
  /*********************************************************************/
 
  // NOTE: Implementing an observer is outside the scope of this post (this example
  // uses a made-up "SampleObserver" to illustrate when/where the observer should 
  // be initialized). 
  
  // The observer could be anything so long as it is able to detect content changes
  // and report them to the loader with a call to onContentChanged(). For example,
  // if you were writing a Loader which loads a list of all installed applications
  // on the device, the observer could be a BroadcastReceiver that listens for the
  // ACTION_PACKAGE_ADDED intent, and calls onContentChanged() on the particular 
  // Loader whenever the receiver detects that a new application has been installed.
  // Please don’t hesitate to leave a comment if you still find this confusing! :)
  private SampleObserver mObserver;
}
</pre>
</p>

<h4>Conclusion</h4>

<p>I hope these posts were useful and gave you a better understanding of how Loaders and the LoaderManager work together to perform asynchronous, auto-updating queries. Remember that Loaders are your friends... if you use them, your app will benefit in both responsiveness and the amount of code you need to write to get everything working properly! Hopefully I could help lessen the learning curve a bit by detailing them out!</p>

<p>As always, please don’t hesitate to <b>leave a comment</b> if you have any questions! And don't forget to +1 this blog in the top right corner if you found it helpful! :)</p>

<hr color='#000000' size='1' width='40%' align='left'/>
<a name="footnote1"><sup>1</sup></a> You don't need to worry about registering a listener for your Loader unless you plan on using it without the LoaderManager. The LoaderManager will act as this "listener" and will forward any results that the Loader delivers to the <code>LoaderCallbacks#onLoadFinished</code> method.<br>
<a name="footnote2"><sup>2</sup></a> Loaders may also be in an <a href="http://developer.android.com/reference/android/content/Loader.html#onAbandon()">“abandoned”</a> state. This is an optional intermediary state between “stopped” and “reset” and is not discussed here for the sake of brevity. That said, in my experience implementing <code>onAbandon()</code> is usually not necessary.