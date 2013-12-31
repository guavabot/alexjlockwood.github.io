---
layout: post
title: Binders & Death Recipients
date: 2013-08-05
permalink: /2013/08/binders-death-recipients.html
comments: true
---

<p><i>Note: before you begin, make sure you've read my <a href="http://www.androiddesignpatterns.com/2013/07/binders-window-tokens.html">previous post</a> about <code>Binder</code> tokens!</i></p>

<p>
Since the very beginning, Android's central focus has been the ability to multitask. In order to achieve it, Android takes a unique approach by allowing multiple applications to run at the same time. Applications are never explicitly closed by the user, but are instead left running at a low priority to be killed by the system when memory is low. This ability to keep processes waiting in the background speeds up app-switching later down the line.
</p>

<p>
Developers learn early on that the key to how Android handles applications in this way is that <b>processes aren't shut down cleanly</b>. Android doesn't rely on applications being well-written and responsive to polite requests to exit. Rather, it brutally force-kills them without warning, allowing the kernel to immediately reclaim resources associated with the process. This helps prevent serious out of memory situations and gives Android total control over misbehaving apps that are negatively impacting the system. For this reason, there is no guarantee that any user-space code (such as an Activity's <code>onDestroy()</code> method) will ever be executed when an application's process goes away.
</p>

<p>
Considering the limited memory available in mobile environments, this approach seems promising. However, there is still one issue that needs to be addressed: <i>how should the system detect an application's death so that it can quickly clean up its state</i>? When an application dies, its state will be spread over dozens of system services (the Activity Manager, Window Manager, Power Manager, etc.) and several different processes. These system services need to be notified immediately when an application dies so that they can clean up its state and maintain an accurate snapshot of the system. Enter death recipients.
</p>

<!--more-->

<h4>Death Recipients</h4>

<p>
As it turns out, this task is made easy using the <code>Binder</code>'s "link-to-death" facility, which allows a process to get a callback when another process hosting a binder object goes away. In Android, any process can receive a notification when another process dies by taking the following steps:</p>

<ol>

<li value="1"><p>First, the process creates a <a href="http://developer.android.com/reference/android/os/IBinder.DeathRecipient.html"><code>DeathRecipient</code></a> callback object containing the code to be executed when the death notification arrives.</p></li>

<li value="2"><p>Next, it obtains a reference to a <code>Binder</code> object that lives in another process and calls its <a href="http://developer.android.com/reference/android/os/Binder.html#linkToDeath(android.os.IBinder.DeathRecipient, int)"><code>linkToDeath(IBinder.DeathRecipient recipient, int flags)</code></a>, passing the <code>DeathRecipient</code> callback object as the first argument.</p></li>

<li value="3"><p>Finally, it waits for the process hosting the <code>Binder</code> object to die. When the Binder kernel driver detects that the process hosting the <code>Binder</code> is gone, it will notify the registered <code>DeathRecipient</code> callback object by calling its <a href="http://developer.android.com/reference/android/os/IBinder.DeathRecipient.html#binderDied()"><code>binderDied()</code></a> method.</p></li>

</ol>

<p>
Analyzing the source code once again gives some insight into how this pattern is used inside the framework. Consider an example application that (similar to the example given in my <a href="http://www.androiddesignpatterns.com/2013/07/binders-window-tokens.html">previous post</a>) acquires a wake lock in <code>onCreate()</code>, but is abruptly killed by the system before it is able to release the wake lock in <code>onDestroy()</code>. How and when will the <a href="https://android.googlesource.com/platform/frameworks/base/+/android-4.3_r2.1/services/java/com/android/server/power/PowerManagerService.java"><code>PowerManagerService</code></a> be notified so that it can quickly release the wake lock before wasting the device's battery? As you might expect, the <code>PowerManagerService</code> achieves this by registering a <code>DeathRecipient</code> (note that some of the source code has been cleaned up for the sake of brevity):
</p>

<p><pre class="brush:java">/**
 * The global power manager system service. Application processes 
 * interact with this class remotely through the PowerManager class.
 *
 * @see frameworks/base/services/java/com/android/server/power/PowerManagerService.java
 */
public final class PowerManagerService extends IPowerManager.Stub {

  // List of all wake locks acquired by applications.
  private List&lt;WakeLock&gt; mWakeLocks = new ArrayList&lt;WakeLock&gt;();

  @Override // Binder call
  public void acquireWakeLock(IBinder token, int flags, String tag) {
    WakeLock wakeLock = new WakeLock(token, flags, tag);

    // Register to receive a notification when the process hosting 
    // the token goes away.
    token.linkToDeath(wakeLock, 0);

    // Acquire the wake lock by adding it as an entry to the list.
    mWakeLocks.add(wakeLock);

    updatePowerState();
  }

  @Override // Binder call
  public void releaseWakeLock(IBinder token, int flags) {
    int index = findWakeLockIndex(token);
    if (index < 0) {
      // The client app has sent us an invalid token, so ignore
      // the request.
      return;
    }

    // Release the wake lock by removing its entry from the list.
    WakeLock wakeLock = mWakeLocks.get(index);
    mWakeLocks.remove(index);

    // We no longer care about receiving death notifications.
    wakeLock.mToken.unlinkToDeath(wakeLock, 0);

    updatePowerState();
  }

  private int findWakeLockIndex(IBinder token) {
    for (int i = 0; i < mWakeLocks.size(); i++) {
      if (mWakeLocks.get(i).mToken == token) {
        return i;
      }
    }
    return -1;
  }

  /**
   * Represents a wake lock that has been acquired by an application.
   */
  private final class WakeLock implements IBinder.DeathRecipient {
    public final IBinder mToken;
    public final int mFlags;
    public final String mTag;

    public WakeLock(IBinder token, inf flags, String tag) {
      mToken = token;
      mFlags = flags;
      mTag = tag;
    }

    @Override
    public void binderDied() {
      int index = mWakeLocks.indexOf(this);
      if (index < 0) {
        return;
      }

      // The token's hosting process was killed before it was
      // able to explicitly release the wake lock, so release 
      // it for them.
      mWakeLocks.remove(index);

      updatePowerState();
    }
  }

  /**
   * Updates the global power state of the device.
   */
  private void updatePowerState() {
    // ...
  }
}</pre></p>

<p>
The code might seem a little dense at first, but the concept is simple:
</p>

<ul>

<li>
<p>
When the application requests to acquire a wake lock, the power manager service's <code>acquireWakeLock()</code> method is called. The power manager service registers the wake lock for the application, and also links to the death of the wake lock's unique <code>Binder</code> token so that it can get notified if the application process is abruptly killed.
</p>
</li>

<li>
<p>
When the application requests to release a wake lock, the power manager service's <code>releaseWakeLock()</code> method is called. The power manager service releases the wake lock for the application, and also unlinks to the death of the wake lock's unique <code>Binder</code> token (as it no longer cares about getting notified when the application process dies).
</p>
</li>

<li>
<p>
When the application is abruptly killed before the wake lock is explicitly released, the Binder kernel driver notices that the wake lock's Binder token has been linked to the death of the application process. The Binder kernel driver quickly dispatches a death notification to the registered death recipient's <code>binderDied()</code> method, which quickly releases the wake lock and updates the device's power state.
</p>
</li>

</ul>

<h4>Examples in the Framework</h4>

<p>
The <code>Binder</code>'s link-to-death feature is an incredibly useful tool that is used extensively by the framework's system services. Here are some of the more interesting examples:
</p>

<ul>

<li>
<p>
The window manager links to the death of the window's <a href="https://developer.android.com/reference/android/view/Window.Callback.html">callback interface</a>. In the rare case that the application's process is killed while its windows are still showing, the window manager will receive a death notification callback, at which point it can clean up after the application by closing its windows.
</p>
</li>

<li>
<p>
When an application binds to a remote service, the application links to the death of the binder stub returned by the remote service's <code>onBind()</code> method. If the remote service suddenly dies, the registered death recipient's <code>binderDied()</code> method is called, which contains some clean up code, as well as the code that calls the <a href="https://developer.android.com/reference/android/content/ServiceConnection.html#onServiceDisconnected(android.content.ComponentName)"><code>onServiceDisconnected(ComponentName name)</code></a> method (the source code illustrating how this is done is located <a href="https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/core/java/android/app/LoadedApk.java">here</a>).
</p>
</li>

<li>
<p>
Many other system services depend on the Binder's link-to-death facility in order to ensure that expensive resources aren't leaked when an application process is forcefully killed. Some other examples (not including the <code>PowerManagerService</code>) are the <a href="https://android.googlesource.com/platform/frameworks/base/+/master/services/java/com/android/server/VibratorService.java"><code>VibratorService</code></a>, <a href="https://android.googlesource.com/platform/frameworks/base/+/master/services/java/com/android/server/LocationManagerService.java"><code>LocationManagerService</code></a>, <a href="https://android.googlesource.com/platform/frameworks/base/+/master/services/java/com/android/server/location/GpsLocationProvider.java"><code>GpsLocationProvider</code></a>, and the <a href="https://android.googlesource.com/platform/frameworks/base/+/master/services/java/com/android/server/wifi/WifiService.java"><code>WifiService</code></a>.
</p>
</li>

</ul>

<h4>Additional Reading</h4>

<p>
If you would like to learn more about <code>Binder</code>s and how they work at a deeper level, I've included some links below. These articles were extremely helpful to me as I was writing these last two posts about <code>Binder</code> tokens and <code>DeathRecipient</code>s, and I would strongly recommend reading them if you get a chance!
</p>

<ul>

<li>
<a href="https://lkml.org/lkml/2009/6/25/3">This post</a> is what initially got me interested in this topic. Special thanks to <a class="g-profile" href="http://plus.google.com/105051985738280261832" target="_blank">+Dianne Hackborn</a> for explaining this!
</li>

<li>
<a href="http://www.nds.rub.de/media/attachments/files/2012/03/binder.pdf">A great paper</a> which explains almost everything you need to know about Binders.
</li>

<li>
<a href="http://events.linuxfoundation.org/images/stories/slides/abs2013_gargentas.pdf">These slides</a> are another great Binder resource.
</li>

<li>
<a href="https://plus.google.com/105051985738280261832/posts/ACaCokiLfqK">This Google+ post</a> talks about how/why live wallpapers are given their own window.
</li>

<li>
<a href="http://android-developers.blogspot.com/2010/04/multitasking-android-way.html">A great blog post</a> discussing multitasking in Android.
</li>

<li>
<a href="https://plus.google.com/105051985738280261832/posts/XAZ4CeVP6DC">This Google+ post</a> talks about how windows are crucial in achieving secure and efficient graphics performance.
</li>

<li>
<a href="http://shop.oreilly.com/product/0636920021094.do">This book</a> taught me a lot about how the application framework works from an embedded systems point of view... and it taught me a couple of really cool <code>adb</code> commands too!
</li>

</ul>

<p>
As always, thanks for reading, and leave a comment if you have any questions. Don't forget to +1 this blog and share this post on Google+ if you found it interesting!
</p>