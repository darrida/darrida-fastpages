---
keywords: fastai
description: A test post to try out fastpages
title: Tranforming and Flattening Data
toc: true
badges: true
comments: true
categories: [csv, defaultdict]
nb_path: _notebooks/2020-06-08-Transforming and Flattening Data.ipynb
layout: notebook
---

<!--
#################################################
### THIS FILE WAS AUTOGENERATED! DO NOT EDIT! ###
#################################################
# file to edit: _notebooks/2020-06-08-Transforming and Flattening Data.ipynb
-->

<div class="container" id="notebook-container">
        
    {% raw %}
    
<div class="cell border-box-sizing code_cell rendered">

</div>
    {% endraw %}

<div class="cell border-box-sizing text_cell rendered"><div class="inner_cell">
<div class="text_cell_render border-box-sizing rendered_html">
<h1 id="The-Situation">The Situation<a class="anchor-link" href="#The-Situation"> </a></h1><p>A while back I had a the need to take what were essentially audit logs, where multiple records existed for each id, and flatten then into a single record for each id.</p>

</div>
</div>
</div>
<div class="cell border-box-sizing text_cell rendered"><div class="inner_cell">
<div class="text_cell_render border-box-sizing rendered_html">
<p>I pulled the data into python from a database as a list of over 100,000 tuples. Here is an example of the data I was looking at.</p>

</div>
</div>
</div>
<div class="cell border-box-sizing text_cell rendered"><div class="inner_cell">
<div class="text_cell_render border-box-sizing rendered_html">

<pre><code>csv
(audit_record.csv)
RECORD,ID,OLD_VALUE,FINAL_VALUE,DATE,TABLE,COLUMN
1, id1, value1,  value2,  02/01/2020, table1, column1
2, id1, value2,  value3,  02/02/2020, table1, column1
3, id1, value3,  value1,  02/03/2020, table1, column1
4, id2, value4,  value5,  02/01/2020, table1, column1
5, id2, value5,  ,        02/03/2020, table1, column1
6, id3, value6,  value7,  02/01/2020, table1, column1
7, id3, value8,  value9,  02/02/2020, table1, column1
8, id4, value10, value11, 02/01/2020, table1, column1
9, id4, value11, value12, 02/02/2020, table1, column1
10,id4, value12, value4,  02/03/2020, table1, column1</code></pre>

</div>
</div>
</div>
<div class="cell border-box-sizing text_cell rendered"><div class="inner_cell">
<div class="text_cell_render border-box-sizing rendered_html">
<h1 id="The-Problem">The Problem<a class="anchor-link" href="#The-Problem"> </a></h1><p>This was an audit log for a series of undesired changes. Here is an example of the progression of these changes:</p>
<ul>
<li>02/01/2020: a large number of records were changed and iterated to the next highest available number</li>
<li>02/02/2020: a similar number of records (but not all) were changed again, again iterating to the next available number</li>
<li>02/03/2020: an additional event took place that resulted in many of the records being corrected (returning to their previous numbers), but some ended at yet another iteration higher, while still others ended up missing a value altogether.</li>
</ul>
<p>In the sample information above there are 4 different unintended transformations of data that occurred:</p>
<p>id1: Example series of changes that ended back at the correct value:</p>

<pre><code>02/01: value1  =&gt;  value2  |  02/02: value2  =&gt;  value3  |  02/03: value3  =&gt;  value1</code></pre>
<p>id2: Example of series of changes that ended with a complete removal of the value:</p>

<pre><code>02/01: value4  =&gt;  value5  |  02/02: No Changes  |  02/03: value5  =&gt;  null</code></pre>
<p>id3: Example of series of changes that ended at a different number:</p>

<pre><code>02/01: value6  =&gt;  value7  |  02/02: value7  =&gt;  value8  |  02/03: No Changes</code></pre>
<p>id4: Example of series of changes that ended up a different id's value (data not only incorrect, but conflicting):</p>

<pre><code>02/01: value10  =&gt;  value11  |  02/02: value11  =&gt;  value12  |  02/03: value12  =&gt;  value4</code></pre>

</div>
</div>
</div>
<div class="cell border-box-sizing text_cell rendered"><div class="inner_cell">
<div class="text_cell_render border-box-sizing rendered_html">
<h2 id="Details">Details<a class="anchor-link" href="#Details"> </a></h2><p><strong>In addition to the obvious differences between these 4 different changes, there are some that are harder to see:</strong></p>
<ul>
<li>Some id records went through 3 changes, others went through 2 (this one is not hard to see)</li>
<li>The last change of some id records was on 02/02, while others' last change was on 02/03</li>
</ul>
<p><strong>Altogether, the different factors that need to be understood are the following:</strong></p>
<ul>
<li>Some records changed each of the 3 days, some only on 2</li>
<li>Some of the records that only changed 2 times had their final changed on day 2, while others skipped day 2 and had their final change on day 3</li>
<li>Some records eventually were self corrected to their original value</li>
<li>Some records iterated 2 times and ended at a different number</li>
<li>Some records eventually ended up with a final value of null</li>
<li>Some records ended up at the original value of a different id</li>
</ul>
<p><strong>This example, as messy as it is, also is cleaner that the situation itself. The follow challenges existed:</strong></p>
<ul>
<li>There were some that only had 1 audit record where the original value was immediately replaced with a null</li>
<li>The list of tuples was not sorted in any fashion</li>
<li>The dates were actually spread over a period of 14 or 15 days, with the changes for a single record following anywhere in that time period -- not a clean 3 days like the example here.</li>
</ul>
<p><strong>The tuples that represent the audits for these changes are all over the place. In the end, what I wanted to see clearly what I was dealing with. In order to do that I needed to have a list of new records that would clearly show me the following for each id:</strong></p>
<ul>
<li>original value, and the date that value was lost</li>
<li>final value, and the date that value was added</li>
</ul>

</div>
</div>
</div>
<div class="cell border-box-sizing text_cell rendered"><div class="inner_cell">
<div class="text_cell_render border-box-sizing rendered_html">
<h1 id="The-Solution">The Solution<a class="anchor-link" href="#The-Solution"> </a></h1><ul>
<li>Gather the change associated with a single id together, identified by that id:</li>
</ul>

</div>
</div>
</div>
    {% raw %}
    
<div class="cell border-box-sizing code_cell rendered">
<div class="input">

<div class="inner_cell">
    <div class="input_area">
<div class=" highlight hl-ipython3"><pre><span></span><span class="kn">import</span> <span class="nn">csv</span>
<span class="kn">from</span> <span class="nn">collections</span> <span class="kn">import</span> <span class="n">defaultdict</span>

<span class="n">reader</span> <span class="o">=</span> <span class="n">csv</span><span class="o">.</span><span class="n">DictReader</span><span class="p">(</span><span class="nb">open</span><span class="p">(</span><span class="s1">&#39;2020-06-08/audit_record.csv&#39;</span><span class="p">))</span>

<span class="n">dict_by_user</span> <span class="o">=</span> <span class="n">defaultdict</span><span class="p">(</span><span class="nb">list</span><span class="p">)</span>
<span class="k">for</span> <span class="n">i</span> <span class="ow">in</span> <span class="n">reader</span><span class="p">:</span>
    <span class="n">dict_by_user</span><span class="p">[</span><span class="n">i</span><span class="p">[</span><span class="s1">&#39;ID&#39;</span><span class="p">]]</span><span class="o">.</span><span class="n">append</span><span class="p">(</span><span class="n">i</span><span class="p">)</span>
</pre></div>

    </div>
</div>
</div>

</div>
    {% endraw %}

<div class="cell border-box-sizing text_cell rendered"><div class="inner_cell">
<div class="text_cell_render border-box-sizing rendered_html">
<ul>
<li>The results in <code>dcit_by_user</code> are the following data structure (python dictionary)</li>
</ul>
<div class="highlight"><pre><span></span><span class="n">dict_by_user</span> <span class="o">=</span> <span class="p">{</span>
    <span class="s1">&#39;id1&#39;</span><span class="p">:</span> <span class="p">[[</span><span class="s1">&#39;id1&#39;</span><span class="p">,</span> <span class="s1">&#39;value1&#39;</span><span class="p">,</span> <span class="s1">&#39;value2&#39;</span><span class="p">,</span>  <span class="s1">&#39;02/01/2020&#39;</span><span class="p">,</span> <span class="s1">&#39;table1&#39;</span><span class="p">,</span> <span class="s1">&#39;column1&#39;</span><span class="p">],</span>
            <span class="p">[</span><span class="s1">&#39;id1&#39;</span><span class="p">,</span> <span class="s1">&#39;value2&#39;</span><span class="p">,</span> <span class="s1">&#39;value3&#39;</span><span class="p">,</span>  <span class="s1">&#39;02/02/2020&#39;</span><span class="p">,</span> <span class="s1">&#39;table1&#39;</span><span class="p">,</span> <span class="s1">&#39;column1&#39;</span><span class="p">],</span>
            <span class="p">[</span><span class="s1">&#39;id1&#39;</span><span class="p">,</span> <span class="s1">&#39;value3&#39;</span><span class="p">,</span> <span class="s1">&#39;value1&#39;</span><span class="p">,</span>  <span class="s1">&#39;02/03/2020&#39;</span><span class="p">,</span> <span class="s1">&#39;table1&#39;</span><span class="p">,</span> <span class="s1">&#39;column1&#39;</span><span class="p">]],</span>
    <span class="s1">&#39;id2&#39;</span><span class="p">:</span> <span class="p">[[</span><span class="s1">&#39;id2&#39;</span><span class="p">,</span> <span class="s1">&#39;value4&#39;</span><span class="p">,</span> <span class="s1">&#39;value5&#39;</span><span class="p">,</span>  <span class="s1">&#39;02/01/2020&#39;</span><span class="p">,</span> <span class="s1">&#39;table1&#39;</span><span class="p">,</span> <span class="s1">&#39;column1&#39;</span><span class="p">],</span>
            <span class="p">[</span><span class="s1">&#39;id2&#39;</span><span class="p">,</span> <span class="s1">&#39;value5&#39;</span><span class="p">,</span>       <span class="s1">&#39;&#39;</span><span class="p">,</span>  <span class="s1">&#39;02/03/2020,&#39;</span> <span class="s1">&#39;table1&#39;</span><span class="p">,</span> <span class="s1">&#39;column1&#39;</span><span class="p">]],</span>
    <span class="s1">&#39;id3&#39;</span><span class="p">:</span> <span class="p">[[</span><span class="s1">&#39;id3&#39;</span><span class="p">,</span> <span class="s1">&#39;value6&#39;</span><span class="p">,</span> <span class="s1">&#39;value7&#39;</span><span class="p">,</span>  <span class="s1">&#39;02/01/2020&#39;</span><span class="p">,</span> <span class="s1">&#39;table1&#39;</span><span class="p">,</span> <span class="s1">&#39;column1&#39;</span><span class="p">],</span>
            <span class="p">[</span><span class="s1">&#39;id3&#39;</span><span class="p">,</span> <span class="s1">&#39;value8&#39;</span><span class="p">,</span> <span class="s1">&#39;value9&#39;</span><span class="p">,</span>  <span class="s1">&#39;02/02/2020&#39;</span><span class="p">,</span> <span class="s1">&#39;table1&#39;</span><span class="p">,</span> <span class="s1">&#39;column1&#39;</span><span class="p">]],</span>
    <span class="s1">&#39;id4&#39;</span><span class="p">:</span> <span class="p">[[</span><span class="s1">&#39;id4&#39;</span><span class="p">,</span> <span class="s1">&#39;value10&#39;</span><span class="p">,</span><span class="s1">&#39;value11&#39;</span><span class="p">,</span> <span class="s1">&#39;02/01/2020&#39;</span><span class="p">,</span> <span class="s1">&#39;table1&#39;</span><span class="p">,</span> <span class="s1">&#39;column1&#39;</span><span class="p">],</span>
            <span class="p">[</span><span class="s1">&#39;id4&#39;</span><span class="p">,</span> <span class="s1">&#39;value11&#39;</span><span class="p">,</span><span class="s1">&#39;value12&#39;</span><span class="p">,</span> <span class="s1">&#39;02/02/2020&#39;</span><span class="p">,</span> <span class="s1">&#39;table1&#39;</span><span class="p">,</span> <span class="s1">&#39;column1&#39;</span><span class="p">],</span>
            <span class="p">[</span><span class="s1">&#39;id4&#39;</span><span class="p">,</span> <span class="s1">&#39;value12&#39;</span><span class="p">,</span><span class="s1">&#39;value4&#39;</span><span class="p">,</span>  <span class="s1">&#39;02/03/2020&#39;</span><span class="p">,</span> <span class="s1">&#39;table1&#39;</span><span class="p">,</span> <span class="s1">&#39;column1&#39;</span><span class="p">]]</span>
<span class="p">}</span>
</pre></div>
<ul>
<li><strong>NOTE:</strong> The actual end result of a defaultdict. The more accurate representation of what the data looks like can be seen by expanding the result right below here.</li>
</ul>

</div>
</div>
</div>
    {% raw %}
    
<div class="cell border-box-sizing code_cell rendered">
<details class="description">
      <summary class="btn btn-sm" data-open="Hide Code" data-close="Show Code"></summary>
        <p><div class="input">

<div class="inner_cell">
    <div class="input_area">
<div class=" highlight hl-ipython3"><pre><span></span><span class="c1">#collapse-hide</span>
<span class="n">dict_by_user</span> <span class="o">=</span> <span class="n">defaultdict</span><span class="p">(</span><span class="o">&lt;</span><span class="k">class</span> <span class="err">&#39;</span><span class="nc">list</span><span class="s1">&#39;&gt;,</span>
                    <span class="p">{</span><span class="s1">&#39; id1&#39;</span><span class="p">:</span> <span class="p">[</span><span class="n">OrderedDict</span><span class="p">([(</span><span class="s1">&#39;RECORD&#39;</span><span class="p">,</span> <span class="s1">&#39;1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;ID&#39;</span><span class="p">,</span> <span class="s1">&#39; id1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;OLD_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;FINAL_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value2&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;DATE&#39;</span><span class="p">,</span> <span class="s1">&#39;02/01/2020&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;TABLE&#39;</span><span class="p">,</span> <span class="s1">&#39; table1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;COLUMN&#39;</span><span class="p">,</span> <span class="s1">&#39; column1&#39;</span><span class="p">)]),</span>
                              <span class="n">OrderedDict</span><span class="p">([(</span><span class="s1">&#39;RECORD&#39;</span><span class="p">,</span> <span class="s1">&#39;2&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;ID&#39;</span><span class="p">,</span> <span class="s1">&#39; id1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;OLD_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value2&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;FINAL_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value3&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;DATE&#39;</span><span class="p">,</span> <span class="s1">&#39;02/02/2020&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;TABLE&#39;</span><span class="p">,</span> <span class="s1">&#39; table1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;COLUMN&#39;</span><span class="p">,</span> <span class="s1">&#39; column1&#39;</span><span class="p">)]),</span>
                              <span class="n">OrderedDict</span><span class="p">([(</span><span class="s1">&#39;RECORD&#39;</span><span class="p">,</span> <span class="s1">&#39;3&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;ID&#39;</span><span class="p">,</span> <span class="s1">&#39; id1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;OLD_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;FINAL_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;DATE&#39;</span><span class="p">,</span> <span class="s1">&#39;02/03/2020&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;TABLE&#39;</span><span class="p">,</span> <span class="s1">&#39; table1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;COLUMN&#39;</span><span class="p">,</span> <span class="s1">&#39; column1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;ORIGINAL_DATE&#39;</span><span class="p">,</span> <span class="s1">&#39;02/01/2020&#39;</span><span class="p">)])],</span>
                     <span class="s1">&#39; id2&#39;</span><span class="p">:</span> <span class="p">[</span><span class="n">OrderedDict</span><span class="p">([(</span><span class="s1">&#39;RECORD&#39;</span><span class="p">,</span> <span class="s1">&#39;4&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;ID&#39;</span><span class="p">,</span> <span class="s1">&#39; id2&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;OLD_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value4&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;FINAL_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value5&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;DATE&#39;</span><span class="p">,</span> <span class="s1">&#39;02/01/2020&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;TABLE&#39;</span><span class="p">,</span> <span class="s1">&#39; table1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;COLUMN&#39;</span><span class="p">,</span> <span class="s1">&#39; column1&#39;</span><span class="p">)]),</span>
                              <span class="n">OrderedDict</span><span class="p">([(</span><span class="s1">&#39;RECORD&#39;</span><span class="p">,</span> <span class="s1">&#39;5&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;ID&#39;</span><span class="p">,</span> <span class="s1">&#39; id2&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;OLD_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value4&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;FINAL_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;DATE&#39;</span><span class="p">,</span> <span class="s1">&#39;02/03/2020&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;TABLE&#39;</span><span class="p">,</span> <span class="s1">&#39; table1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;COLUMN&#39;</span><span class="p">,</span> <span class="s1">&#39; column1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;ORIGINAL_DATE&#39;</span><span class="p">,</span> <span class="s1">&#39;02/01/2020&#39;</span><span class="p">)])],</span>
                     <span class="s1">&#39; id3&#39;</span><span class="p">:</span> <span class="p">[</span><span class="n">OrderedDict</span><span class="p">([(</span><span class="s1">&#39;RECORD&#39;</span><span class="p">,</span> <span class="s1">&#39;6&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;ID&#39;</span><span class="p">,</span> <span class="s1">&#39; id3&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;OLD_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value6&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;FINAL_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value7&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;DATE&#39;</span><span class="p">,</span> <span class="s1">&#39;02/01/2020&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;TABLE&#39;</span><span class="p">,</span> <span class="s1">&#39; table1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;COLUMN&#39;</span><span class="p">,</span> <span class="s1">&#39; column1&#39;</span><span class="p">)]),</span>
                              <span class="n">OrderedDict</span><span class="p">([(</span><span class="s1">&#39;RECORD&#39;</span><span class="p">,</span> <span class="s1">&#39;7&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;ID&#39;</span><span class="p">,</span> <span class="s1">&#39; id3&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;OLD_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value6&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;FINAL_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value9&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;DATE&#39;</span><span class="p">,</span> <span class="s1">&#39;02/02/2020&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;TABLE&#39;</span><span class="p">,</span> <span class="s1">&#39; table1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;COLUMN&#39;</span><span class="p">,</span> <span class="s1">&#39; column1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;ORIGINAL_DATE&#39;</span><span class="p">,</span> <span class="s1">&#39;02/01/2020&#39;</span><span class="p">)])],</span>
                     <span class="s1">&#39; id4&#39;</span><span class="p">:</span> <span class="p">[</span><span class="n">OrderedDict</span><span class="p">([(</span><span class="s1">&#39;RECORD&#39;</span><span class="p">,</span> <span class="s1">&#39;8&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;ID&#39;</span><span class="p">,</span> <span class="s1">&#39; id4&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;OLD_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value10&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;FINAL_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value11&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;DATE&#39;</span><span class="p">,</span> <span class="s1">&#39;02/01/2020&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;TABLE&#39;</span><span class="p">,</span> <span class="s1">&#39; table1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;COLUMN&#39;</span><span class="p">,</span> <span class="s1">&#39; column1&#39;</span><span class="p">)]),</span>
                              <span class="n">OrderedDict</span><span class="p">([(</span><span class="s1">&#39;RECORD&#39;</span><span class="p">,</span> <span class="s1">&#39;9&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;ID&#39;</span><span class="p">,</span> <span class="s1">&#39; id4&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;OLD_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value10&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;FINAL_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value12&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;DATE&#39;</span><span class="p">,</span> <span class="s1">&#39;02/02/2020&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;TABLE&#39;</span><span class="p">,</span> <span class="s1">&#39; table1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;COLUMN&#39;</span><span class="p">,</span> <span class="s1">&#39; column1&#39;</span><span class="p">),</span>
                                           <span class="p">(</span><span class="s1">&#39;ORIGINAL_DATE&#39;</span><span class="p">,</span> <span class="s1">&#39;02/01/2020&#39;</span><span class="p">)])],</span>
                     <span class="s1">&#39;id4&#39;</span><span class="p">:</span> <span class="p">[</span><span class="n">OrderedDict</span><span class="p">([(</span><span class="s1">&#39;RECORD&#39;</span><span class="p">,</span> <span class="s1">&#39;10&#39;</span><span class="p">),</span>
                                          <span class="p">(</span><span class="s1">&#39;ID&#39;</span><span class="p">,</span> <span class="s1">&#39;id4&#39;</span><span class="p">),</span>
                                          <span class="p">(</span><span class="s1">&#39;OLD_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value12&#39;</span><span class="p">),</span>
                                          <span class="p">(</span><span class="s1">&#39;FINAL_VALUE&#39;</span><span class="p">,</span> <span class="s1">&#39;value4&#39;</span><span class="p">),</span>
                                          <span class="p">(</span><span class="s1">&#39;DATE&#39;</span><span class="p">,</span> <span class="s1">&#39;02/03/2020&#39;</span><span class="p">),</span>
                                          <span class="p">(</span><span class="s1">&#39;TABLE&#39;</span><span class="p">,</span> <span class="s1">&#39; table1&#39;</span><span class="p">),</span>
                                          <span class="p">(</span><span class="s1">&#39;COLUMN&#39;</span><span class="p">,</span> <span class="s1">&#39; column1&#39;</span><span class="p">),</span>
                                          <span class="p">(</span><span class="s1">&#39;ORIGINAL_DATE&#39;</span><span class="p">,</span> <span class="s1">&#39;02/03/2020&#39;</span><span class="p">)])]})</span>
</pre></div>

    </div>
</div>
</div>
</p>
    </details>
</div>
    {% endraw %}

<div class="cell border-box-sizing text_cell rendered"><div class="inner_cell">
<div class="text_cell_render border-box-sizing rendered_html">
<ul>
<li><code>dict_by_user</code> is then passed into the next section:</li>
</ul>

</div>
</div>
</div>
    {% raw %}
    
<div class="cell border-box-sizing code_cell rendered">
<div class="input">

<div class="inner_cell">
    <div class="input_area">
<div class=" highlight hl-ipython3"><pre><span></span><span class="n">audit_summary_l</span> <span class="o">=</span> <span class="p">[]</span>
<span class="k">for</span> <span class="n">i</span> <span class="ow">in</span> <span class="n">dict_by_user</span><span class="p">:</span>
    <span class="k">if</span> <span class="n">dict_by_user</span><span class="p">:</span>
        <span class="n">temp_dict</span> <span class="o">=</span> <span class="p">{}</span>
        <span class="nb">max</span> <span class="o">=</span> <span class="s1">&#39;0&#39;</span>
        <span class="nb">min</span> <span class="o">=</span> <span class="s1">&#39;99/99/9999&#39;</span>
        <span class="k">for</span> <span class="n">record</span> <span class="ow">in</span> <span class="n">dict_by_user</span><span class="p">[</span><span class="n">i</span><span class="p">]:</span>
            <span class="k">if</span> <span class="n">record</span><span class="p">[</span><span class="s1">&#39;DATE&#39;</span><span class="p">]</span> <span class="o">&gt;</span> <span class="nb">max</span><span class="p">:</span>
                <span class="nb">max</span> <span class="o">=</span> <span class="n">record</span><span class="p">[</span><span class="s1">&#39;DATE&#39;</span><span class="p">]</span>
                <span class="n">last</span> <span class="o">=</span> <span class="n">record</span>
            <span class="k">if</span> <span class="n">record</span><span class="p">[</span><span class="s1">&#39;DATE&#39;</span><span class="p">]</span> <span class="o">&lt;</span> <span class="nb">min</span><span class="p">:</span>
                <span class="nb">min</span> <span class="o">=</span> <span class="n">record</span><span class="p">[</span><span class="s1">&#39;DATE&#39;</span><span class="p">]</span>
                <span class="n">first</span> <span class="o">=</span> <span class="n">record</span>
        <span class="n">temp_dict</span> <span class="o">=</span> <span class="n">last</span>
        <span class="n">temp_dict</span><span class="p">[</span><span class="s1">&#39;OLD_VALUE&#39;</span><span class="p">]</span> <span class="o">=</span> <span class="n">first</span><span class="p">[</span><span class="s1">&#39;OLD_VALUE&#39;</span><span class="p">]</span>
        <span class="n">temp_dict</span><span class="p">[</span><span class="s1">&#39;ORIGINAL_DATE&#39;</span><span class="p">]</span> <span class="o">=</span> <span class="n">first</span><span class="p">[</span><span class="s1">&#39;DATE&#39;</span><span class="p">]</span>
        <span class="n">audit_summary_l</span><span class="o">.</span><span class="n">append</span><span class="p">(</span><span class="n">temp_dict</span><span class="p">)</span>

<span class="n">columns</span> <span class="o">=</span> <span class="p">[</span><span class="s1">&#39;ID&#39;</span><span class="p">,</span><span class="s1">&#39;OLD_VALUE&#39;</span><span class="p">,</span><span class="s1">&#39;ORIGINAL_DATE&#39;</span><span class="p">,</span><span class="s1">&#39;FINAL_VALUE&#39;</span><span class="p">,</span>
           <span class="s1">&#39;DATE&#39;</span><span class="p">,</span><span class="s1">&#39;TABLE&#39;</span><span class="p">,</span><span class="s1">&#39;COLUMN&#39;</span><span class="p">,</span><span class="s1">&#39;RECORD&#39;</span><span class="p">]</span>
<span class="k">with</span> <span class="nb">open</span><span class="p">(</span><span class="s1">&#39;2020-06-08/audit_summary.csv&#39;</span><span class="p">,</span> <span class="s1">&#39;w&#39;</span><span class="p">)</span> <span class="k">as</span> <span class="n">output_file</span><span class="p">:</span>
    <span class="n">dict_writer</span> <span class="o">=</span> <span class="n">csv</span><span class="o">.</span><span class="n">DictWriter</span><span class="p">(</span><span class="n">output_file</span><span class="p">,</span> <span class="n">fieldnames</span><span class="o">=</span><span class="n">columns</span><span class="p">,</span> <span class="n">lineterminator</span><span class="o">=</span><span class="s1">&#39;</span><span class="se">\n</span><span class="s1">&#39;</span><span class="p">)</span>
    <span class="n">dict_writer</span><span class="o">.</span><span class="n">writeheader</span><span class="p">()</span>
    <span class="k">for</span> <span class="n">data</span> <span class="ow">in</span> <span class="n">audit_summary_l</span><span class="p">:</span>
        <span class="n">dict_writer</span><span class="o">.</span><span class="n">writerow</span><span class="p">(</span><span class="n">data</span><span class="p">)</span>
</pre></div>

    </div>
</div>
</div>

</div>
    {% endraw %}

<div class="cell border-box-sizing text_cell rendered"><div class="inner_cell">
<div class="text_cell_render border-box-sizing rendered_html">
<ul>
<li>This results in the following csv data:</li>
</ul>

<pre><code>csv
ID,OLD_VALUE,ORIGINAL_DATE,FINAL_VALUE,DATE,TABLE,COLUMN,RECORD
id1, value1,  02/01/2020, value1, 02/03/2020, table1,column1, 3
id2, value4,  02/01/2020,       , 02/03/2020, table1,column1, 5
id3, value6,  02/01/2020, value9, 02/02/2020, table1,column1, 7
id4, value10, 02/01/2020, value4, 02/03/2020,table 1,column1, 10</code></pre>

</div>
</div>
</div>
</div>
 
