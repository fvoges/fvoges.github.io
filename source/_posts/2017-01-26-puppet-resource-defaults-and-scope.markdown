---
layout: post
title: "Puppet resource defaults and scope"
date: 2017-01-26 15:59:56 +0000
comments: true
categories: puppet
published: true
---
Recently, someone asked me this in HipChat:

{% blockquote %}
@boats suppose you have resource defaults in example.pp and resource defaults in init.pp, if you include example.pp in init.pp, does resource default in init.pp take over resource defaults in example.pp?
{% endblockquote %}

The answer to that is in the official Puppet documentation. To be precise, it's in this page [Language: Resource default statements](https://docs.puppet.com/puppet/latest/lang_defaults.html).

But my answer was "it's complicated :D" (yeh, my typical kind of answer)

Why? Because the result depends on how you're using those classes.

## The simple case

For a start, the resource defaults affect the local scope and anything underneath it. In other words, it can also affect resources declared inside classes declared within that scope.

If you have these two classes,

{% codeblock lang:Puppet resource_defaults_test/manifests/init.pp %}
class resource_defaults_test {
  File {
    owner => 'user1',
    group => 'user1',
    mode =>  '0600',
  }

  include resource_defaults_test::subclass
}
{% endcodeblock %}

and

{% codeblock lang:Puppet resource_defaults_test/manifests/subclass.pp %}
class resource_defaults_test::subclass {
  File {
    owner => 'user2',
    group => 'user2',
  }

  file { '/tmp/resources_defaults_test':
    ensure => 'file',
  }
}
{% endcodeblock %}

The file resource in the `resource_defaults_test::subclass` class inherits the resource defaults from `resource_defaults_test`. That means that it will use the merged resource defaults from `resource_defaults_test::subclass` and `resource_defaults_test`. In this case, the `owner` and `group` attribute values from the current scope and the `mode` attribute value from the parent scope.

## The not so simple case

But what happens if you also declare `resource_defaults_test::subclass` somewhere else in your code?

That's where the answer becomes 'it's complicated'. Although I should probably say 'it depends'.

Why? Because in that case it will depend on which class declaration is parsed first.

### Example 1
{% codeblock lang:Puppet example 1 %}

include resource_defaults_test
{% endcodeblock %}


