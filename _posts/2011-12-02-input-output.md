---
layout: post
title: Input and output in Adhearsion 2.0
tags: [input, output, adhearsion, punchblock, prism]
last_updated: 2011-12-02
---

The ability to play audio to the users and receive input for them is
vital for a voice application.
Existing functionality has been rebuilt and updated to keep
compatibility with the previous methods, while enhancing it with the
innovations it needs to compete in a growing market.


{% highlight ruby %}
{% endhighlight %}

Output
------

Punchblock offers support for a variety of engines that do not only
provide audio files output, but use TTS to allow developers to provide
a more engaging caller experience.
Voxeo PRISM incorporates its own media server, while Asterisk will need
to be used together with an SSML capable TTS engine such as UniMRCP.
In the case Asterisk does not have an SSML component installed, the
ahn-asterisk gem provides an interface to the basic functionalities.

The core output methods in Adhearsion have been expanded to
auto-detect various types of arguments. It will be further
augmented by a work-in-progress plugin system that will allow developers
to specify their own additional output types to detect and play.

The basic method for rendering audio into a channel is *play*
{% highlight ruby %}
# dialplan.rb
play 'Good morning and welcome'
play 1
play Time.now, :strftime => '%H%m' # Time and Date objects accept options to control the format to play
play '/app/sounds/hello.mp3', :fallback => 'Hello' # Specifying a TTS fallback for an audio file
{% endhighlight %}

Internally, auto-detection of output is handled by the *detect_type*
method. If the application needs specific functionality that is not
handled correctly by *play*, the methods *play_text*, *play_numeric*,
*play_time* and *play_audio* provide a first level of fine-grained
control.

If a developer requires even more specific output, the full capabilites
of SSML can be used through the RubySpeech library to achieve complete
control on TTS.

{% highlight ruby %}
ssml = RubySpeech::SSML.draw do
   string "Please press a button"
end
play ssml
{% endhighlight %}

Input
-----

