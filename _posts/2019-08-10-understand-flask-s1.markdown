---
layout: post
title:  "Understand Flask (一)"
date:   2019-08-10 09:15:39 +0000
categories: code
tail: ""
description: "<em>how</em> Flask and Werkzeug do what they do with these context locals."
---

<div class="post-text" itemprop="text">
<p>Previous answers already give a nice overview of what goes on in the background of Flask during a request. If you haven't read it yet I recommend @MarkHildreth's answer prior to reading this. In short, a new context (thread) is created for each http request, which is why it's necessary to have a thread <code>Local</code> facility that allows objects such as <code>request</code> and <code>g</code> to be accessible globally across threads, while maintaining their request specific context. Furthermore, while processing an http request Flask can emulate additional requests from within, hence the necessity to store their respective context on a stack. Also, Flask allows multiple wsgi applications to run along each other within a single process, and more than one can be called to action  during a request (each request creates a new application context), hence the need for a context stack for applications. That's a summary of what was covered in previous answers.</p>

<p>My goal now is to complement our current understanding by explaining <em>how</em> Flask and Werkzeug do what they do with these context locals. I simplified the code to enhance the understanding of its logic, but if you get this, you should be able to easily grasp most of what's in the actual source (<code>werkzeug.local</code> and <code>flask.globals</code>).</p>

<p>Let's first understand how Werkzeug implements thread Locals. </p>

<h1>Local</h1>

<p>When an http request comes in, it is processed within the context of a single thread. As an alternative mean to spawn a new context during an http request, Werkzeug also allows the use of greenlets (a sort of lighter "micro-threads") instead of normal threads. If you don't have greenlets installed it will revert to using threads instead. Each of these threads (or greenlets) are identifiable by a unique id, which you can retrieve with the module's <code>get_ident()</code> function. That function is the starting point to the magic behind having <code>request</code>, <code>current_app</code>,<code>url_for</code>, <code>g</code>, and other such context-bound global objects.</p>

<pre class="lang-py prettyprint prettyprinted" style=""><code><span class="kwd">try</span><span class="pun">:</span><span class="pln">
    </span><span class="kwd">from</span><span class="pln"> greenlet </span><span class="kwd">import</span><span class="pln"> get_ident
</span><span class="kwd">except</span><span class="pln"> </span><span class="typ">ImportError</span><span class="pun">:</span><span class="pln">
    </span><span class="kwd">from</span><span class="pln"> thread </span><span class="kwd">import</span><span class="pln"> get_ident</span></code></pre>

<p>Now that we have our identity function we can know which thread we're on at any given time and we can create what's called a thread <code>Local</code>, a contextual object that can be accessed globally, but when you access its attributes they resolve to their value for that specific thread.
e.g.</p>

<pre class="lang-py prettyprint prettyprinted" style=""><code><span class="com"># globally</span><span class="pln">
local </span><span class="pun">=</span><span class="pln"> </span><span class="typ">Local</span><span class="pun">()</span><span class="pln">

</span><span class="com"># ...</span><span class="pln">

</span><span class="com"># on thread 1</span><span class="pln">
local</span><span class="pun">.</span><span class="pln">first_name </span><span class="pun">=</span><span class="pln"> </span><span class="str">'John'</span><span class="pln">

</span><span class="com"># ...</span><span class="pln">

</span><span class="com"># on thread 2</span><span class="pln">
local</span><span class="pun">.</span><span class="pln">first_name </span><span class="pun">=</span><span class="pln"> </span><span class="str">'Debbie'</span></code></pre>

<p>Both values are present on the globally accessible <code>Local</code> object at the same time, but accessing <code>local.first_name</code> within the context of thread 1 will give you <code>'John'</code>, whereas it will return <code>'Debbie'</code> on thread 2.</p>

<p>How is that possible? Let's look at some (simplified) code:</p>

<pre class="lang-py prettyprint prettyprinted" style=""><code><span class="kwd">class</span><span class="pln"> </span><span class="typ">Local</span><span class="pun">(</span><span class="pln">object</span><span class="pun">)</span><span class="pln">
    </span><span class="kwd">def</span><span class="pln"> __init__</span><span class="pun">(</span><span class="pln">self</span><span class="pun">):</span><span class="pln">
        self</span><span class="pun">.</span><span class="pln">storage </span><span class="pun">=</span><span class="pln"> </span><span class="pun">{}</span><span class="pln">

    </span><span class="kwd">def</span><span class="pln"> __getattr__</span><span class="pun">(</span><span class="pln">self</span><span class="pun">,</span><span class="pln"> name</span><span class="pun">):</span><span class="pln">
        context_id </span><span class="pun">=</span><span class="pln"> get_ident</span><span class="pun">()</span><span class="pln"> </span><span class="com"># we get the current thread's or greenlet's id</span><span class="pln">
        contextual_storage </span><span class="pun">=</span><span class="pln"> self</span><span class="pun">.</span><span class="pln">storage</span><span class="pun">.</span><span class="pln">setdefault</span><span class="pun">(</span><span class="pln">context_id</span><span class="pun">,</span><span class="pln"> </span><span class="pun">{})</span><span class="pln">
        </span><span class="kwd">try</span><span class="pun">:</span><span class="pln">
            </span><span class="kwd">return</span><span class="pln"> contextual_storage</span><span class="pun">[</span><span class="pln">name</span><span class="pun">]</span><span class="pln">
        </span><span class="kwd">except</span><span class="pln"> </span><span class="typ">KeyError</span><span class="pun">:</span><span class="pln">
            </span><span class="kwd">raise</span><span class="pln"> </span><span class="typ">AttributeError</span><span class="pun">(</span><span class="pln">name</span><span class="pun">)</span><span class="pln">

    </span><span class="kwd">def</span><span class="pln"> __setattr__</span><span class="pun">(</span><span class="pln">self</span><span class="pun">,</span><span class="pln"> name</span><span class="pun">,</span><span class="pln"> value</span><span class="pun">):</span><span class="pln">
        context_id </span><span class="pun">=</span><span class="pln"> get_ident</span><span class="pun">()</span><span class="pln">
        contextual_storage </span><span class="pun">=</span><span class="pln"> self</span><span class="pun">.</span><span class="pln">storage</span><span class="pun">.</span><span class="pln">setdefault</span><span class="pun">(</span><span class="pln">context_id</span><span class="pun">,</span><span class="pln"> </span><span class="pun">{})</span><span class="pln">
        contextual_storage</span><span class="pun">[</span><span class="pln">name</span><span class="pun">]</span><span class="pln"> </span><span class="pun">=</span><span class="pln"> value

    </span><span class="kwd">def</span><span class="pln"> __release_local__</span><span class="pun">(</span><span class="pln">self</span><span class="pun">):</span><span class="pln">
        context_id </span><span class="pun">=</span><span class="pln"> get_ident</span><span class="pun">()</span><span class="pln">
        self</span><span class="pun">.</span><span class="pln">storage</span><span class="pun">.</span><span class="pln">pop</span><span class="pun">(</span><span class="pln">context_id</span><span class="pun">,</span><span class="pln"> </span><span class="kwd">None</span><span class="pun">)</span><span class="pln">

local </span><span class="pun">=</span><span class="pln"> </span><span class="typ">Local</span><span class="pun">()</span></code></pre>

<p>From the code above we can see that the magic boils down to <code>get_ident()</code> which identifies the current greenlet or thread. The <code>Local</code> storage then just uses that as a key to store any data contextual to the current thread.</p>

<p>You can have multiple <code>Local</code> objects per process and <code>request</code>, <code>g</code>, <code>current_app</code> and others could simply have been created like that. But that's not how it's done in Flask in which these are not <em>technically</em> <code>Local</code> objects, but more accurately <code>LocalProxy</code> objects. What's a <code>LocalProxy</code>? </p>

<h1>LocalProxy</h1>

<p>A LocalProxy is an object that queries a <code>Local</code> to find another object of interest (i.e. the object it proxies to). Let's take a look to understand:</p>

<pre class="lang-py prettyprint prettyprinted" style=""><code><span class="kwd">class</span><span class="pln"> </span><span class="typ">LocalProxy</span><span class="pun">(</span><span class="pln">object</span><span class="pun">):</span><span class="pln">
    </span><span class="kwd">def</span><span class="pln"> __init__</span><span class="pun">(</span><span class="pln">self</span><span class="pun">,</span><span class="pln"> local</span><span class="pun">,</span><span class="pln"> name</span><span class="pun">):</span><span class="pln">
        </span><span class="com"># `local` here is either an actual `Local` object, that can be used</span><span class="pln">
        </span><span class="com"># to find the object of interest, here identified by `name`, or it's</span><span class="pln">
        </span><span class="com"># a callable that can resolve to that proxied object</span><span class="pln">
        self</span><span class="pun">.</span><span class="pln">local </span><span class="pun">=</span><span class="pln"> local
        </span><span class="com"># `name` is an identifier that will be passed to the local to find the</span><span class="pln">
        </span><span class="com"># object of interest.</span><span class="pln">
        self</span><span class="pun">.</span><span class="pln">name </span><span class="pun">=</span><span class="pln"> name

    </span><span class="kwd">def</span><span class="pln"> _get_current_object</span><span class="pun">(</span><span class="pln">self</span><span class="pun">):</span><span class="pln">
        </span><span class="com"># if `self.local` is truly a `Local` it means that it implements</span><span class="pln">
        </span><span class="com"># the `__release_local__()` method which, as its name implies, is</span><span class="pln">
        </span><span class="com"># normally used to release the local. We simply look for it here</span><span class="pln">
        </span><span class="com"># to identify which is actually a Local and which is rather just</span><span class="pln">
        </span><span class="com"># a callable:</span><span class="pln">
        </span><span class="kwd">if</span><span class="pln"> hasattr</span><span class="pun">(</span><span class="pln">self</span><span class="pun">.</span><span class="pln">local</span><span class="pun">,</span><span class="pln"> </span><span class="str">'__release_local__'</span><span class="pun">):</span><span class="pln">
            </span><span class="kwd">try</span><span class="pun">:</span><span class="pln">
                </span><span class="kwd">return</span><span class="pln"> getattr</span><span class="pun">(</span><span class="pln">self</span><span class="pun">.</span><span class="pln">local</span><span class="pun">,</span><span class="pln"> self</span><span class="pun">.</span><span class="pln">name</span><span class="pun">)</span><span class="pln">
            </span><span class="kwd">except</span><span class="pln"> </span><span class="typ">AttributeError</span><span class="pun">:</span><span class="pln">
                </span><span class="kwd">raise</span><span class="pln"> </span><span class="typ">RuntimeError</span><span class="pun">(</span><span class="str">'no object bound to %s'</span><span class="pln"> </span><span class="pun">%</span><span class="pln"> self</span><span class="pun">.</span><span class="pln">name</span><span class="pun">)</span><span class="pln">

        </span><span class="com"># if self.local is not actually a Local it must be a callable that </span><span class="pln">
        </span><span class="com"># would resolve to the object of interest.</span><span class="pln">
        </span><span class="kwd">return</span><span class="pln"> self</span><span class="pun">.</span><span class="pln">local</span><span class="pun">(</span><span class="pln">self</span><span class="pun">.</span><span class="pln">name</span><span class="pun">)</span><span class="pln">

    </span><span class="com"># Now for the LocalProxy to perform its intended duties i.e. proxying </span><span class="pln">
    </span><span class="com"># to an underlying object located somewhere in a Local, we turn all magic</span><span class="pln">
    </span><span class="com"># methods into proxies for the same methods in the object of interest.</span><span class="pln">
    </span><span class="lit">@property</span><span class="pln">
    </span><span class="kwd">def</span><span class="pln"> __dict__</span><span class="pun">(</span><span class="pln">self</span><span class="pun">):</span><span class="pln">
        </span><span class="kwd">try</span><span class="pun">:</span><span class="pln">
            </span><span class="kwd">return</span><span class="pln"> self</span><span class="pun">.</span><span class="pln">_get_current_object</span><span class="pun">().</span><span class="pln">__dict__
        </span><span class="kwd">except</span><span class="pln"> </span><span class="typ">RuntimeError</span><span class="pun">:</span><span class="pln">
            </span><span class="kwd">raise</span><span class="pln"> </span><span class="typ">AttributeError</span><span class="pun">(</span><span class="str">'__dict__'</span><span class="pun">)</span><span class="pln">

    </span><span class="kwd">def</span><span class="pln"> __repr__</span><span class="pun">(</span><span class="pln">self</span><span class="pun">):</span><span class="pln">
        </span><span class="kwd">try</span><span class="pun">:</span><span class="pln">
            </span><span class="kwd">return</span><span class="pln"> repr</span><span class="pun">(</span><span class="pln">self</span><span class="pun">.</span><span class="pln">_get_current_object</span><span class="pun">())</span><span class="pln">
        </span><span class="kwd">except</span><span class="pln"> </span><span class="typ">RuntimeError</span><span class="pun">:</span><span class="pln">
            </span><span class="kwd">return</span><span class="pln"> </span><span class="str">'&lt;%s unbound&gt;'</span><span class="pln"> </span><span class="pun">%</span><span class="pln"> self</span><span class="pun">.</span><span class="pln">__class__</span><span class="pun">.</span><span class="pln">__name__

    </span><span class="kwd">def</span><span class="pln"> __bool__</span><span class="pun">(</span><span class="pln">self</span><span class="pun">):</span><span class="pln">
        </span><span class="kwd">try</span><span class="pun">:</span><span class="pln">
            </span><span class="kwd">return</span><span class="pln"> bool</span><span class="pun">(</span><span class="pln">self</span><span class="pun">.</span><span class="pln">_get_current_object</span><span class="pun">())</span><span class="pln">
        </span><span class="kwd">except</span><span class="pln"> </span><span class="typ">RuntimeError</span><span class="pun">:</span><span class="pln">
            </span><span class="kwd">return</span><span class="pln"> </span><span class="kwd">False</span><span class="pln">

    </span><span class="com"># ... etc etc ... </span><span class="pln">

    </span><span class="kwd">def</span><span class="pln"> __getattr__</span><span class="pun">(</span><span class="pln">self</span><span class="pun">,</span><span class="pln"> name</span><span class="pun">):</span><span class="pln">
        </span><span class="kwd">if</span><span class="pln"> name </span><span class="pun">==</span><span class="pln"> </span><span class="str">'__members__'</span><span class="pun">:</span><span class="pln">
            </span><span class="kwd">return</span><span class="pln"> dir</span><span class="pun">(</span><span class="pln">self</span><span class="pun">.</span><span class="pln">_get_current_object</span><span class="pun">())</span><span class="pln">
        </span><span class="kwd">return</span><span class="pln"> getattr</span><span class="pun">(</span><span class="pln">self</span><span class="pun">.</span><span class="pln">_get_current_object</span><span class="pun">(),</span><span class="pln"> name</span><span class="pun">)</span><span class="pln">

    </span><span class="kwd">def</span><span class="pln"> __setitem__</span><span class="pun">(</span><span class="pln">self</span><span class="pun">,</span><span class="pln"> key</span><span class="pun">,</span><span class="pln"> value</span><span class="pun">):</span><span class="pln">
        self</span><span class="pun">.</span><span class="pln">_get_current_object</span><span class="pun">()[</span><span class="pln">key</span><span class="pun">]</span><span class="pln"> </span><span class="pun">=</span><span class="pln"> value

    </span><span class="kwd">def</span><span class="pln"> __delitem__</span><span class="pun">(</span><span class="pln">self</span><span class="pun">,</span><span class="pln"> key</span><span class="pun">):</span><span class="pln">
        </span><span class="kwd">del</span><span class="pln"> self</span><span class="pun">.</span><span class="pln">_get_current_object</span><span class="pun">()[</span><span class="pln">key</span><span class="pun">]</span><span class="pln">

    </span><span class="com"># ... and so on ...</span><span class="pln">

    __setattr__ </span><span class="pun">=</span><span class="pln"> </span><span class="kwd">lambda</span><span class="pln"> x</span><span class="pun">,</span><span class="pln"> n</span><span class="pun">,</span><span class="pln"> v</span><span class="pun">:</span><span class="pln"> setattr</span><span class="pun">(</span><span class="pln">x</span><span class="pun">.</span><span class="pln">_get_current_object</span><span class="pun">(),</span><span class="pln"> n</span><span class="pun">,</span><span class="pln"> v</span><span class="pun">)</span><span class="pln">
    __delattr__ </span><span class="pun">=</span><span class="pln"> </span><span class="kwd">lambda</span><span class="pln"> x</span><span class="pun">,</span><span class="pln"> n</span><span class="pun">:</span><span class="pln"> delattr</span><span class="pun">(</span><span class="pln">x</span><span class="pun">.</span><span class="pln">_get_current_object</span><span class="pun">(),</span><span class="pln"> n</span><span class="pun">)</span><span class="pln">
    __str__ </span><span class="pun">=</span><span class="pln"> </span><span class="kwd">lambda</span><span class="pln"> x</span><span class="pun">:</span><span class="pln"> str</span><span class="pun">(</span><span class="pln">x</span><span class="pun">.</span><span class="pln">_get_current_object</span><span class="pun">())</span><span class="pln">
    __lt__ </span><span class="pun">=</span><span class="pln"> </span><span class="kwd">lambda</span><span class="pln"> x</span><span class="pun">,</span><span class="pln"> o</span><span class="pun">:</span><span class="pln"> x</span><span class="pun">.</span><span class="pln">_get_current_object</span><span class="pun">()</span><span class="pln"> </span><span class="pun">&lt;</span><span class="pln"> o
    __le__ </span><span class="pun">=</span><span class="pln"> </span><span class="kwd">lambda</span><span class="pln"> x</span><span class="pun">,</span><span class="pln"> o</span><span class="pun">:</span><span class="pln"> x</span><span class="pun">.</span><span class="pln">_get_current_object</span><span class="pun">()</span><span class="pln"> </span><span class="pun">&lt;=</span><span class="pln"> o
    __eq__ </span><span class="pun">=</span><span class="pln"> </span><span class="kwd">lambda</span><span class="pln"> x</span><span class="pun">,</span><span class="pln"> o</span><span class="pun">:</span><span class="pln"> x</span><span class="pun">.</span><span class="pln">_get_current_object</span><span class="pun">()</span><span class="pln"> </span><span class="pun">==</span><span class="pln"> o

    </span><span class="com"># ... and so forth ...</span></code></pre>

<p>Now to create globally accessible proxies you would do </p>

<pre class="lang-py prettyprint prettyprinted" style=""><code><span class="com"># this would happen some time near application start-up</span><span class="pln">
local </span><span class="pun">=</span><span class="pln"> </span><span class="typ">Local</span><span class="pun">()</span><span class="pln">
request </span><span class="pun">=</span><span class="pln"> </span><span class="typ">LocalProxy</span><span class="pun">(</span><span class="pln">local</span><span class="pun">,</span><span class="pln"> </span><span class="str">'request'</span><span class="pun">)</span><span class="pln">
g </span><span class="pun">=</span><span class="pln"> </span><span class="typ">LocalProxy</span><span class="pun">(</span><span class="pln">local</span><span class="pun">,</span><span class="pln"> </span><span class="str">'g'</span><span class="pun">)</span></code></pre>

<p>and now some time early over the course of a request you would store some objects inside the local that the previously created proxies can access, no matter which thread we're on</p>

<pre class="lang-py prettyprint prettyprinted" style=""><code><span class="com"># this would happen early during processing of an http request</span><span class="pln">
local</span><span class="pun">.</span><span class="pln">request </span><span class="pun">=</span><span class="pln"> </span><span class="typ">RequestContext</span><span class="pun">(</span><span class="pln">http_environment</span><span class="pun">)</span><span class="pln">
local</span><span class="pun">.</span><span class="pln">g </span><span class="pun">=</span><span class="pln"> </span><span class="typ">SomeGeneralPurposeContainer</span><span class="pun">()</span></code></pre>

<p>The advantage of using <code>LocalProxy</code> as globally accessible objects rather than making them <code>Locals</code> themselves is that it simplifies their management. You only just need a single <code>Local</code> object to create many globally accessible proxies. At the end of the request, during cleanup, you simply release the one <code>Local</code> (i.e. you pop the context_id from its storage) and don't bother with the proxies, they're still globally accessible and still defer to the one <code>Local</code> to find their object of interest for subsequent http requests.</p>

<pre class="lang-py prettyprint prettyprinted" style=""><code><span class="com"># this would happen some time near the end of request processing</span><span class="pln">
release</span><span class="pun">(</span><span class="pln">local</span><span class="pun">)</span><span class="pln"> </span><span class="com"># aka local.__release_local__()</span></code></pre>

<p>To simplify the creation of a <code>LocalProxy</code> when we already have a <code>Local</code>, Werkzeug implements the <code>Local.__call__()</code> magic method as follows:</p>

<pre class="lang-py prettyprint prettyprinted" style=""><code><span class="kwd">class</span><span class="pln"> </span><span class="typ">Local</span><span class="pun">(</span><span class="pln">object</span><span class="pun">):</span><span class="pln">
    </span><span class="com"># ... </span><span class="pln">
    </span><span class="com"># ... all same stuff as before go here ...</span><span class="pln">
    </span><span class="com"># ... </span><span class="pln">

    </span><span class="kwd">def</span><span class="pln"> __call__</span><span class="pun">(</span><span class="pln">self</span><span class="pun">,</span><span class="pln"> name</span><span class="pun">):</span><span class="pln">
        </span><span class="kwd">return</span><span class="pln"> </span><span class="typ">LocalProxy</span><span class="pun">(</span><span class="pln">self</span><span class="pun">,</span><span class="pln"> name</span><span class="pun">)</span><span class="pln">

</span><span class="com"># now you can do</span><span class="pln">
local </span><span class="pun">=</span><span class="pln"> </span><span class="typ">Local</span><span class="pun">()</span><span class="pln">
request </span><span class="pun">=</span><span class="pln"> local</span><span class="pun">(</span><span class="str">'request'</span><span class="pun">)</span><span class="pln">
g </span><span class="pun">=</span><span class="pln"> local</span><span class="pun">(</span><span class="str">'g'</span><span class="pun">)</span></code></pre>

<p>However, if you look in the Flask source (flask.globals) that's still not how <code>request</code>, <code>g</code>, <code>current_app</code> and <code>session</code> are created. As we've established, Flask can spawn multiple "fake" requests (from a single true http request) and in the process also push multiple application contexts. This isn't a common use-case, but it's a capability of the framework. Since these "concurrent" requests and apps are still limited to run with only one having the "focus" at any time, it makes sense to use a stack for their respective context. Whenever a new request is spawned or one of the applications is called, they push their context at the top of their respective stack. Flask uses <code>LocalStack</code> objects for this purpose. When they conclude their business they pop the context out of the stack.</p>

<h1>LocalStack</h1>

<p>This is what a <code>LocalStack</code> looks like (again the code is simplified to facilitate understanding of its logic).</p>

<pre class="lang-py prettyprint prettyprinted" style=""><code><span class="kwd">class</span><span class="pln"> </span><span class="typ">LocalStack</span><span class="pun">(</span><span class="pln">object</span><span class="pun">):</span><span class="pln">

    </span><span class="kwd">def</span><span class="pln"> __init__</span><span class="pun">(</span><span class="pln">self</span><span class="pun">):</span><span class="pln">
        self</span><span class="pun">.</span><span class="pln">local </span><span class="pun">=</span><span class="pln"> </span><span class="typ">Local</span><span class="pun">()</span><span class="pln">

    </span><span class="kwd">def</span><span class="pln"> push</span><span class="pun">(</span><span class="pln">self</span><span class="pun">,</span><span class="pln"> obj</span><span class="pun">):</span><span class="pln">
        </span><span class="str">"""Pushes a new item to the stack"""</span><span class="pln">
        rv </span><span class="pun">=</span><span class="pln"> getattr</span><span class="pun">(</span><span class="pln">self</span><span class="pun">.</span><span class="pln">local</span><span class="pun">,</span><span class="pln"> </span><span class="str">'stack'</span><span class="pun">,</span><span class="pln"> </span><span class="kwd">None</span><span class="pun">)</span><span class="pln">
        </span><span class="kwd">if</span><span class="pln"> rv </span><span class="kwd">is</span><span class="pln"> </span><span class="kwd">None</span><span class="pun">:</span><span class="pln">
            self</span><span class="pun">.</span><span class="pln">local</span><span class="pun">.</span><span class="pln">stack </span><span class="pun">=</span><span class="pln"> rv </span><span class="pun">=</span><span class="pln"> </span><span class="pun">[]</span><span class="pln">
        rv</span><span class="pun">.</span><span class="pln">append</span><span class="pun">(</span><span class="pln">obj</span><span class="pun">)</span><span class="pln">
        </span><span class="kwd">return</span><span class="pln"> rv

    </span><span class="kwd">def</span><span class="pln"> pop</span><span class="pun">(</span><span class="pln">self</span><span class="pun">):</span><span class="pln">
        </span><span class="str">"""Removes the topmost item from the stack, will return the
        old value or `None` if the stack was already empty.
        """</span><span class="pln">
        stack </span><span class="pun">=</span><span class="pln"> getattr</span><span class="pun">(</span><span class="pln">self</span><span class="pun">.</span><span class="pln">local</span><span class="pun">,</span><span class="pln"> </span><span class="str">'stack'</span><span class="pun">,</span><span class="pln"> </span><span class="kwd">None</span><span class="pun">)</span><span class="pln">
        </span><span class="kwd">if</span><span class="pln"> stack </span><span class="kwd">is</span><span class="pln"> </span><span class="kwd">None</span><span class="pun">:</span><span class="pln">
            </span><span class="kwd">return</span><span class="pln"> </span><span class="kwd">None</span><span class="pln">
        </span><span class="kwd">elif</span><span class="pln"> len</span><span class="pun">(</span><span class="pln">stack</span><span class="pun">)</span><span class="pln"> </span><span class="pun">==</span><span class="pln"> </span><span class="lit">1</span><span class="pun">:</span><span class="pln">
            release_local</span><span class="pun">(</span><span class="pln">self</span><span class="pun">.</span><span class="pln">local</span><span class="pun">)</span><span class="pln"> </span><span class="com"># this simply releases the local</span><span class="pln">
            </span><span class="kwd">return</span><span class="pln"> stack</span><span class="pun">[-</span><span class="lit">1</span><span class="pun">]</span><span class="pln">
        </span><span class="kwd">else</span><span class="pun">:</span><span class="pln">
            </span><span class="kwd">return</span><span class="pln"> stack</span><span class="pun">.</span><span class="pln">pop</span><span class="pun">()</span><span class="pln">

    </span><span class="lit">@property</span><span class="pln">
    </span><span class="kwd">def</span><span class="pln"> top</span><span class="pun">(</span><span class="pln">self</span><span class="pun">):</span><span class="pln">
        </span><span class="str">"""The topmost item on the stack.  If the stack is empty,
        `None` is returned.
        """</span><span class="pln">
        </span><span class="kwd">try</span><span class="pun">:</span><span class="pln">
            </span><span class="kwd">return</span><span class="pln"> self</span><span class="pun">.</span><span class="pln">local</span><span class="pun">.</span><span class="pln">stack</span><span class="pun">[-</span><span class="lit">1</span><span class="pun">]</span><span class="pln">
        </span><span class="kwd">except</span><span class="pln"> </span><span class="pun">(</span><span class="typ">AttributeError</span><span class="pun">,</span><span class="pln"> </span><span class="typ">IndexError</span><span class="pun">):</span><span class="pln">
            </span><span class="kwd">return</span><span class="pln"> </span><span class="kwd">None</span></code></pre>

<p>Note from the above that a <code>LocalStack</code> is a stack stored in a local, not a bunch of locals stored on a stack. This implies that although the stack is globally accessible it's a different stack in each thread.</p>

<p>Flask doesn't have its <code>request</code>, <code>current_app</code>, <code>g</code>, and <code>session</code> objects resolving directly to a <code>LocalStack</code>, it rather uses <code>LocalProxy</code> objects that wrap a lookup function (instead of a <code>Local</code> object) that will find the underlying object from the <code>LocalStack</code>:</p>

<pre class="lang-py prettyprint prettyprinted" style=""><code><span class="pln">_request_ctx_stack </span><span class="pun">=</span><span class="pln"> </span><span class="typ">LocalStack</span><span class="pun">()</span><span class="pln">
</span><span class="kwd">def</span><span class="pln"> _find_request</span><span class="pun">():</span><span class="pln">
    top </span><span class="pun">=</span><span class="pln"> _request_ctx_stack</span><span class="pun">.</span><span class="pln">top
    </span><span class="kwd">if</span><span class="pln"> top </span><span class="kwd">is</span><span class="pln"> </span><span class="kwd">None</span><span class="pun">:</span><span class="pln">
        </span><span class="kwd">raise</span><span class="pln"> </span><span class="typ">RuntimeError</span><span class="pun">(</span><span class="str">'working outside of request context'</span><span class="pun">)</span><span class="pln">
    </span><span class="kwd">return</span><span class="pln"> top</span><span class="pun">.</span><span class="pln">request
request </span><span class="pun">=</span><span class="pln"> </span><span class="typ">LocalProxy</span><span class="pun">(</span><span class="pln">_find_request</span><span class="pun">)</span><span class="pln">

</span><span class="kwd">def</span><span class="pln"> _find_session</span><span class="pun">():</span><span class="pln">
    top </span><span class="pun">=</span><span class="pln"> _request_ctx_stack</span><span class="pun">.</span><span class="pln">top
    </span><span class="kwd">if</span><span class="pln"> top </span><span class="kwd">is</span><span class="pln"> </span><span class="kwd">None</span><span class="pun">:</span><span class="pln">
        </span><span class="kwd">raise</span><span class="pln"> </span><span class="typ">RuntimeError</span><span class="pun">(</span><span class="str">'working outside of request context'</span><span class="pun">)</span><span class="pln">
    </span><span class="kwd">return</span><span class="pln"> top</span><span class="pun">.</span><span class="pln">session
session </span><span class="pun">=</span><span class="pln"> </span><span class="typ">LocalProxy</span><span class="pun">(</span><span class="pln">_find_session</span><span class="pun">)</span><span class="pln">

_app_ctx_stack </span><span class="pun">=</span><span class="pln"> </span><span class="typ">LocalStack</span><span class="pun">()</span><span class="pln">
</span><span class="kwd">def</span><span class="pln"> _find_g</span><span class="pun">():</span><span class="pln">
    top </span><span class="pun">=</span><span class="pln"> _app_ctx_stack</span><span class="pun">.</span><span class="pln">top
    </span><span class="kwd">if</span><span class="pln"> top </span><span class="kwd">is</span><span class="pln"> </span><span class="kwd">None</span><span class="pun">:</span><span class="pln">
        </span><span class="kwd">raise</span><span class="pln"> </span><span class="typ">RuntimeError</span><span class="pun">(</span><span class="str">'working outside of application context'</span><span class="pun">)</span><span class="pln">
    </span><span class="kwd">return</span><span class="pln"> top</span><span class="pun">.</span><span class="pln">g
g </span><span class="pun">=</span><span class="pln"> </span><span class="typ">LocalProxy</span><span class="pun">(</span><span class="pln">_find_g</span><span class="pun">)</span><span class="pln">

</span><span class="kwd">def</span><span class="pln"> _find_app</span><span class="pun">():</span><span class="pln">
    top </span><span class="pun">=</span><span class="pln"> _app_ctx_stack</span><span class="pun">.</span><span class="pln">top
    </span><span class="kwd">if</span><span class="pln"> top </span><span class="kwd">is</span><span class="pln"> </span><span class="kwd">None</span><span class="pun">:</span><span class="pln">
        </span><span class="kwd">raise</span><span class="pln"> </span><span class="typ">RuntimeError</span><span class="pun">(</span><span class="str">'working outside of application context'</span><span class="pun">)</span><span class="pln">
    </span><span class="kwd">return</span><span class="pln"> top</span><span class="pun">.</span><span class="pln">app
current_app </span><span class="pun">=</span><span class="pln"> </span><span class="typ">LocalProxy</span><span class="pun">(</span><span class="pln">_find_app</span><span class="pun">)</span></code></pre>

<p>All these are declared at application start-up, but do not actually resolve to anything until a request context or application context is pushed to their respective stack.</p>

<p>If you're curious to see how a context is actually inserted in the stack (and subsequently popped out), look in <code>flask.app.Flask.wsgi_app()</code> which is the point of entry of the wsgi app (i.e. what the web server calls and pass the http environment to when a request comes in), and follow the creation of the <code>RequestContext</code> object all through its subsequent <code>push()</code> into <code>_request_ctx_stack</code>. Once pushed at the top of the stack, it's accessible via <code>_request_ctx_stack.top</code>. Here's some abbreviated code to demonstrate the flow:</p>

<p>So you start an app and make it available to the WSGI server...</p>

<pre class="lang-py prettyprint prettyprinted" style=""><code><span class="pln">app </span><span class="pun">=</span><span class="pln"> </span><span class="typ">Flask</span><span class="pun">(*</span><span class="pln">config</span><span class="pun">,</span><span class="pln"> </span><span class="pun">**</span><span class="pln">kwconfig</span><span class="pun">)</span><span class="pln">

</span><span class="com"># ...</span></code></pre>

<p>Later an http request comes in and the WSGI server calls the app with the usual params...</p>

<pre class="lang-py prettyprint prettyprinted" style=""><code><span class="pln">app</span><span class="pun">(</span><span class="pln">environ</span><span class="pun">,</span><span class="pln"> start_response</span><span class="pun">)</span><span class="pln"> </span><span class="com"># aka app.__call__(environ, start_response)</span></code></pre>

<p>This is roughly what happens in the app...</p>

<pre class="lang-py prettyprint prettyprinted" style=""><code><span class="kwd">def</span><span class="pln"> </span><span class="typ">Flask</span><span class="pun">(</span><span class="pln">object</span><span class="pun">):</span><span class="pln">

    </span><span class="com"># ...</span><span class="pln">

    </span><span class="kwd">def</span><span class="pln"> __call__</span><span class="pun">(</span><span class="pln">self</span><span class="pun">,</span><span class="pln"> environ</span><span class="pun">,</span><span class="pln"> start_response</span><span class="pun">):</span><span class="pln">
        </span><span class="kwd">return</span><span class="pln"> self</span><span class="pun">.</span><span class="pln">wsgi_app</span><span class="pun">(</span><span class="pln">environ</span><span class="pun">,</span><span class="pln"> start_response</span><span class="pun">)</span><span class="pln">

    </span><span class="kwd">def</span><span class="pln"> wsgi_app</span><span class="pun">(</span><span class="pln">self</span><span class="pun">,</span><span class="pln"> environ</span><span class="pun">,</span><span class="pln"> start_response</span><span class="pun">):</span><span class="pln">
        ctx </span><span class="pun">=</span><span class="pln"> </span><span class="typ">RequestContext</span><span class="pun">(</span><span class="pln">self</span><span class="pun">,</span><span class="pln"> environ</span><span class="pun">)</span><span class="pln">
        ctx</span><span class="pun">.</span><span class="pln">push</span><span class="pun">()</span><span class="pln">
        </span><span class="kwd">try</span><span class="pun">:</span><span class="pln">
            </span><span class="com"># process the request here</span><span class="pln">
            </span><span class="com"># raise error if any</span><span class="pln">
            </span><span class="com"># return Response</span><span class="pln">
        </span><span class="kwd">finally</span><span class="pun">:</span><span class="pln">
            ctx</span><span class="pun">.</span><span class="pln">pop</span><span class="pun">()</span><span class="pln">

    </span><span class="com"># ...</span></code></pre>

<p>and this is roughly what happens with RequestContext...</p>

<pre class="lang-py prettyprint prettyprinted" style=""><code><span class="kwd">class</span><span class="pln"> </span><span class="typ">RequestContext</span><span class="pun">(</span><span class="pln">object</span><span class="pun">):</span><span class="pln">

    </span><span class="kwd">def</span><span class="pln"> __init__</span><span class="pun">(</span><span class="pln">self</span><span class="pun">,</span><span class="pln"> app</span><span class="pun">,</span><span class="pln"> environ</span><span class="pun">,</span><span class="pln"> request</span><span class="pun">=</span><span class="kwd">None</span><span class="pun">):</span><span class="pln">
        self</span><span class="pun">.</span><span class="pln">app </span><span class="pun">=</span><span class="pln"> app
        </span><span class="kwd">if</span><span class="pln"> request </span><span class="kwd">is</span><span class="pln"> </span><span class="kwd">None</span><span class="pun">:</span><span class="pln">
            request </span><span class="pun">=</span><span class="pln"> app</span><span class="pun">.</span><span class="pln">request_class</span><span class="pun">(</span><span class="pln">environ</span><span class="pun">)</span><span class="pln">
        self</span><span class="pun">.</span><span class="pln">request </span><span class="pun">=</span><span class="pln"> request
        self</span><span class="pun">.</span><span class="pln">url_adapter </span><span class="pun">=</span><span class="pln"> app</span><span class="pun">.</span><span class="pln">create_url_adapter</span><span class="pun">(</span><span class="pln">self</span><span class="pun">.</span><span class="pln">request</span><span class="pun">)</span><span class="pln">
        self</span><span class="pun">.</span><span class="pln">session </span><span class="pun">=</span><span class="pln"> self</span><span class="pun">.</span><span class="pln">app</span><span class="pun">.</span><span class="pln">open_session</span><span class="pun">(</span><span class="pln">self</span><span class="pun">.</span><span class="pln">request</span><span class="pun">)</span><span class="pln">
        </span><span class="kwd">if</span><span class="pln"> self</span><span class="pun">.</span><span class="pln">session </span><span class="kwd">is</span><span class="pln"> </span><span class="kwd">None</span><span class="pun">:</span><span class="pln">
            self</span><span class="pun">.</span><span class="pln">session </span><span class="pun">=</span><span class="pln"> self</span><span class="pun">.</span><span class="pln">app</span><span class="pun">.</span><span class="pln">make_null_session</span><span class="pun">()</span><span class="pln">
        self</span><span class="pun">.</span><span class="pln">flashes </span><span class="pun">=</span><span class="pln"> </span><span class="kwd">None</span><span class="pln">

    </span><span class="kwd">def</span><span class="pln"> push</span><span class="pun">(</span><span class="pln">self</span><span class="pun">):</span><span class="pln">
        _request_ctx_stack</span><span class="pun">.</span><span class="pln">push</span><span class="pun">(</span><span class="pln">self</span><span class="pun">)</span><span class="pln">

    </span><span class="kwd">def</span><span class="pln"> pop</span><span class="pun">(</span><span class="pln">self</span><span class="pun">):</span><span class="pln">
        _request_ctx_stack</span><span class="pun">.</span><span class="pln">pop</span><span class="pun">()</span></code></pre>

<p>Say a request has finished initializing, the lookup for <code>request.path</code> from one of your view functions would therefore go as follow:</p>

<ul>
<li>start from the globally accessible <code>LocalProxy</code> object <code>request</code>.</li>
<li>to find its underlying object of interest (the object it's proxying to) it calls its lookup function <code>_find_request()</code> (the function it registered as its <code>self.local</code>).</li>
<li>that function queries the <code>LocalStack</code> object <code>_request_ctx_stack</code> for the top context on the stack.</li>
<li>to find the top context, the <code>LocalStack</code> object first queries its inner <code>Local</code> attribute (<code>self.local</code>) for the <code>stack</code> property that was previously stored there.</li>
<li>from the <code>stack</code> it gets the top context</li>
<li>and <code>top.request</code> is thus resolved as the underlying object of interest.</li>
<li>from that object we get the <code>path</code> attribute</li>
</ul>

<p>So we've seen how <code>Local</code>, <code>LocalProxy</code>, and <code>LocalStack</code> work, now think for a moment of the implications and nuances in retrieving the <code>path</code> from:</p>

<ul>
<li>a <code>request</code> object that would be a simple globally accessible object.</li>
<li>a <code>request</code> object that would be a local.</li>
<li>a <code>request</code> object stored as an attribute of a local.</li>
<li>a <code>request</code> object that is a proxy to an object stored in a local.</li>
<li>a <code>request</code> object stored on a stack, that is in turn stored in a local.</li>
<li>a <code>request</code> object that is a proxy to an object on a stack stored in a local. &lt;- this is what Flask does.</li>
</ul>
    </div>
