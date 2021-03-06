---
layout: post
title: 'Handling Configuration Changes with Fragments'
date: 2013-04-29
permalink: /2013/04/retaining-objects-across-config-changes.html
related: ['/2013/08/fragment-transaction-commit-state-loss.html',
          '/2013/01/inner-class-handler-memory-leak.html',
          '/2013/04/activitys-threads-memory-leaks.html']
thumbnails: ['/assets/images/posts/2013/04/29/worker-fragments-screenshot.png']
updated: '2014-01-14'
---
This post addresses a common question that is frequently asked on [StackOverflow](http://stackoverflow.com/q/3821423/844882):

> What is the best way to retain active objects&mdash;such as
> running `Thread`s, `Socket`s, and `AsyncTask`s&mdash;across
> device configuration changes?

To answer this question, we will first discuss some of the common
difficulties developers face when using long-running background tasks
in conjunction with the Activity lifecycle. Then, we will describe
the flaws of two common approaches to solving the problem. Finally,
we will conclude with sample code illustrating the recommended
solution, which uses retained Fragments to achieve our goal.

<!--more-->

### Configuration Changes & Background Tasks

One problem with configuration changes and the destroy-and-create cycle
that Activitys go through as a result stems from the fact that these events
are unpredictable and may occur at any time. Concurrent background tasks
only add to this problem. Assume, for example, that an Activity starts
an `AsyncTask` and soon after the user rotates the screen, causing the
Activity to be destroyed and recreated. When the `AsyncTask` eventually
finishes its work, it will incorrectly report its results back to the
old Activity instance, completely unaware that a new Activity has been
created. As if this wasn't already an issue, the new Activity instance
might waste valuable resources by firing up the background work _again_,
unaware that the old `AsyncTask` is still running. For these reasons,
it is vital that we correctly and efficiently retain active objects
across Activity instances when configuration changes occur.

### Bad Practice: Retain the Activity

Perhaps the hackiest and most widely abused workaround is to disable
the default destroy-and-recreate behavior by setting the `android:configChanges`
attribute in your Android manifest. The apparent simplicity of this
approach makes it extremely attractive to developers;
<a href="http://stackoverflow.com/a/5336057/844882">Google engineers</a>,
however, discourage its use. The primary concern is that it requires you
to handle device configuration changes manually in code. Handling
configuration changes requires you to take many additional steps to
ensure that each and every string, layout, drawable, dimension, etc.
remains in sync with the device's current configuration, and if you
aren't careful, your application can easily have a whole series of
resource-specific bugs as a result.

Another reason why Google discourages its use is because many
developers incorrectly assume that setting `android:configChanges="orientation"`
(for example) will magically protect their application from
unpredictable scenarios in which the underlying Activity will be
destroyed and recreated. _This is not the case._ Configuration
changes can occur for a number of reasons&mdash;not just screen
orientation changes. Inserting your device into a display dock,
changing the default language, and modifying the device's default
font scaling factor are just three examples of events that can
trigger a device configuration change, all of which signal the
system to destroy and recreate all currently running Activitys
the next time they are resumed. As a result, setting the
`android:configChanges` attribute is generally not good practice.

### Deprecated: Override `onRetainNonConfigurationInstance()`

Prior to Honeycomb's release, the recommended means of transferring
active objects across Activity instances was to override the
`onRetainNonConfigurationInstance()` and `getLastNonConfigurationInstance()`
methods. Using this approach, transferring an active object
across Activity instances was merely a matter of returning the
active object in `onRetainNonConfigurationInstance()` and retrieving
it in `getLastNonConfigurationInstance()`. As of API 13, these methods
have been deprecated in favor of the more Fragment's `setRetainInstance(boolean)`
capability, which provides a much cleaner and modular means of
retaining objects during configuration changes. We discuss this
Fragment-based approach in the next section.

### Recommended: Manage the Object Inside a Retained `Fragment`

Ever since the introduction of Fragments in Android 3.0, the recommended
means of retaining active objects across Activity instances is to wrap
and manage them inside of a retained "worker" Fragment. By default,
Fragments are destroyed and recreated along with their parent Activitys
when a configuration change occurs. Calling `Fragment#setRetainInstance(true)`
allows us to bypass this destroy-and-recreate cycle, signaling the system to
retain the current instance of the fragment when the activity is recreated.
As we will see, this will prove to be extremely useful with Fragments that
hold objects like running `Thread`s, `AsyncTask`s, `Socket`s, etc.

The sample code below serves as a basic example of how to retain an
`AsyncTask` across a configuration change using retained Fragments.
The code guarantees that progress updates and results are delivered
back to the currently displayed Activity instance and ensures that
we never accidentally leak an `AsyncTask` during a configuration change.
The design consists of two classes, a `MainActivity`...

<div class="scrollable">
{% highlight java linenos=table %}
/**
 * This Activity displays the screen's UI, creates a TaskFragment
 * to manage the task, and receives progress updates and results 
 * from the TaskFragment when they occur.
 */
public class MainActivity extends Activity implements TaskFragment.TaskCallbacks {

  private TaskFragment mTaskFragment;

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.main);

    FragmentManager fm = getFragmentManager();
    mTaskFragment = (TaskFragment) fm.findFragmentByTag("task");

    // If the Fragment is non-null, then it is currently being
    // retained across a configuration change.
    if (mTaskFragment == null) {
      mTaskFragment = new TaskFragment();
      fm.beginTransaction().add(mTaskFragment, "task").commit();
    }

    // TODO: initialize views, restore saved state, etc.
  }

  // The four methods below are called by the TaskFragment when new
  // progress updates or results are available. The MainActivity 
  // should respond by updating its UI to indicate the change.

  @Override
  public void onPreExecute() { ... }

  @Override
  public void onProgressUpdate(int percent) { ... }

  @Override
  public void onCancelled() { ... }

  @Override
  public void onPostExecute() { ... }
}
{% endhighlight %}
</div>

...and a `TaskFragment`...

<div class="scrollable">
{% highlight java linenos=table %}
/**
 * This Fragment manages a single background task and retains 
 * itself across configuration changes.
 */
public class TaskFragment extends Fragment {

  /**
   * Callback interface through which the fragment will report the
   * task's progress and results back to the Activity.
   */
  public static interface TaskCallbacks {
    void onPreExecute();
    void onProgressUpdate(int percent);
    void onCancelled();
    void onPostExecute();
  }

  private TaskCallbacks mCallbacks;
  private DummyTask mTask;

  /**
   * Hold a reference to the parent Activity so we can report the
   * task's current progress and results. The Android framework 
   * will pass us a reference to the newly created Activity after 
   * each configuration change.
   */
  @Override
  public void onAttach(Activity activity) {
    super.onAttach(activity);
    mCallbacks = (TaskCallbacks) activity;
  }

  /**
   * This method will only be called once when the retained
   * Fragment is first created.
   */
  @Override
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    // Retain this fragment across configuration changes.
    setRetainInstance(true);

    // Create and execute the background task.
    mTask = new DummyTask();
    mTask.execute();
  }

  /**
   * Set the callback to null so we don't accidentally leak the 
   * Activity instance.
   */
  @Override
  public void onDetach() {
    super.onDetach();
    mCallbacks = null;
  }

  /**
   * A dummy task that performs some (dumb) background work and
   * proxies progress updates and results back to the Activity.
   *
   * Note that we need to check if the callbacks are null in each
   * method in case they are invoked after the Activity's and
   * Fragment's onDestroy() method have been called.
   */
  private class DummyTask extends AsyncTask<Void, Integer, Void> {

    @Override
    protected void onPreExecute() {
      if (mCallbacks != null) {
        mCallbacks.onPreExecute();
      }
    }

    /**
     * Note that we do NOT call the callback object's methods
     * directly from the background thread, as this could result 
     * in a race condition.
     */
    @Override
    protected Void doInBackground(Void... ignore) {
      for (int i = 0; !isCancelled() && i < 100; i++) {
        SystemClock.sleep(100);
        publishProgress(i);
      }
      return null;
    }

    @Override
    protected void onProgressUpdate(Integer... percent) {
      if (mCallbacks != null) {
        mCallbacks.onProgressUpdate(percent[0]);
      }
    }

    @Override
    protected void onCancelled() {
      if (mCallbacks != null) {
        mCallbacks.onCancelled();
      }
    }

    @Override
    protected void onPostExecute(Void ignore) {
      if (mCallbacks != null) {
        mCallbacks.onPostExecute();
      }
    }
  }
}
{% endhighlight %}
</div>

### Flow of Events

When the `MainActivity` starts up for the first time, it instantiates and adds
the `TaskFragment` to the Activity's state. The `TaskFragment` creates and
executes an `AsyncTask` and proxies progress updates and results back to the
`MainActivity` via the `TaskCallbacks` interface. When a configuration change
occurs, the `MainActivity` goes through its normal lifecycle events, and once
created the new Activity instance is passed to the `onAttach(Activity)` method,
thus ensuring that the `TaskFragment` will always hold a reference to the
currently displayed Activity instance even after the configuration change.
The resulting design is both simple and reliable; the application framework
will handle re-assigning Activity instances as they are torn down and recreated,
and the `TaskFragment` and its `AsyncTask` never need to worry about the
unpredictable occurrence of a configuration change. Note also that it is impossible
for `onPostExecute()` to be executed in between the calls to `onDetach()` and
`onAttach()`, as explained in <a href="http://stackoverflow.com/q/19964180/844882">this StackOverflow answer</a>
and in my reply to Doug Stevenson in
<a href="https://plus.google.com/u/0/+AlexLockwood/posts/etWuiiRiqLf">this Google+ post</a>
(there is also some discussion about this in the comments below).

### Conclusion

Synchronizing background tasks with the Activity lifecycle can be tricky and
configuration changes will only add to the confusion. Fortunately, retained
Fragments make handling these events very easy by consistently maintaining a
reference to its parent Activity, even after being destroyed and recreated.

A sample application illustrating how to correctly use retained Fragments to
achieve this effect is available for download on the
<a href="https://play.google.com/store/apps/details?id=com.adp.retaintask">Play Store</a>.
The source code is available on <a href="https://github.com/alexjlockwood/worker-fragments">GitHub</a>.
Download it, import it into Eclipse, and modify it all you want!

<a href="/assets/images/posts/2013/04/29/worker-fragments-screenshot.png">
<img src="/assets/images/posts/2013/04/29/worker-fragments-screenshot.png" style="max-width:400px;height=225px;" />
</a>

As always, leave a comment if you have any questions and don't forget to +1 this
blog in the top right corner!
