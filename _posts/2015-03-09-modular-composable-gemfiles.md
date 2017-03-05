---
layout: post
title: Modular, composable Gemfiles
---

TL;DR If you think the size of the Gemfile in your app has got out of control you can group gems into themed sets, put them in smaller gemfiles and include them using eval_gemfile. Smaller gemfiles can also be used for speeding up subsets of the test suite.

Even if you are new to Ruby development it's a no-brainer that the place to add new libraries to your app is the Gemfile. After running `bundle install` your Gemfile.lock will be updated and it will now contain libraries you chose together with their dependencies.

# eval_gemfile

I have seen some truly modular Ruby applications. Most of them were Rails applications that utilised Rails Engines. It's quite a nice mechanism that allows you to create small, specialised applications and place them in a larger context. Each engine comes with its own Gemfile and tests that you can run in isolation.

Modularity is not an easy task in itself, but Rails Engines are quite easy to use. Not all Ruby applications are Rails applications though. Yet they can still be modular.

`eval_gemfile` is not very popular feature of the Bundler DSL (that's the actual name of the domain specific language you use in Gemfiles). In a nutshell, it will add the contents of a gemfile to another one. When you bundle the following gemfile:

```
source "https://rubygems.org"
eval_gemfile('GemfileFinances')

gem 'sinatra'

```

the contents of the file named GemfileFinances will be added to the currently executed gemfile:

```
gem 'finance'
gem 'money'
```

Imagine you had to divide a self-contained part of your application responsible for handling finances. It would be sensible then to define its dependencies separately from the rest. GemfileFinances sounds like a good candidate here! Of course you could still keep dependencies of all parts of your application in one gemfile, but it might not always be the best way.

If our modules are really self-contained we probably want to be able to test them in isolation and run only the necessary set of dependencies. Why would we do that? To save our time mostly. I rarely had problems with the speed of test suite on any Sinatra app, but I cannot say the same thing about Rails apps. So if I have a Rails app, should I use Rails Engines then? It's entirely up to you. Note that not every application component needs to be a small Rails application.

Let's have a look how we could run tests for financial module only. Whenever executing any bundle command, Bundler will normally look for Gemfile. Using non-default gemfile requires setting up an environment variable BUNDLE_GEMFILE whose value is the name of the gemfile:

$ BUNDLE_GEMFILE=GemfileFinancial bundle exec rake test:financial

# Gems source

If you look at both examples carefully, you will notice that I only defined gems source in the main gemfile. GemfileFinancial has no information about where to download gems from. Bundler is clever enough to look for gems on the host machine first BUT we are in trouble if they haven't been installed yet and we still want to run isolated test for our modules. One recipe for this problem is to have an extra gemfile with "Sourced" suffix in the name which specifies the source and requires the one with dependencies:

```
source "https://rubygems.org"
eval_gemfile('GemfileFinancial')
```

Then tests should execute without any errors (at least in theory ;-)).:

$ BUNDLE_GEMFILE=GemfileFinancialSourced bundle exec rake test:financial

It is possible, although not recommended in Bundler 1.7 onwards, to use multiple global sources. They will be searched from last to first. Not ideal and can lead to gem conflicts.

You can use source blocks though:

source 'https://rubygems.org' do
  gem 'finance'
  gem 'money'
end

This removes the need of having extra gemfile specifying source and also gives you the flexibility to use different gems source in each gemfile.

# Common dependencies

What about gems that are in common among modules? It's very likely that even self-contained modules will use the same dependencies and they cannot be mutually exclusive (your app won't start otherwise). It's quite handy to have a separate file defining common dependencies and require it inside of more specific gemfile. Purists could say that your module tests will lose a bit of dependency isolation, but it shouldn't be a problem. You can always put `require: false` for each entry and require them in right places only. :-)

# Should I use it?

There has been lot of discussion about  solving problem of monolith applications recently. The solution you hear about most of the time is microservices architecture. Another approach is to extract parts of the application into gems. None of them is a silver bullet. Using many gemfiles can cause minor confusion as the majority of apps will have only one Gemfile, but if you want to achieve the goal of modularisation while still keeping all code in one place then you should consider it as one available option.

A couple of suitable uses cases for employing the multiple Gemfile's approach include temporarily splitting out Gemfiles during a Rails upgrade, and as part of the extraction process of pulling a gem out of a Sinatra application.

The first use case was applied to the update of a huge Rails 3.2 application to 4.1. We separated dependencies into three groups: common and specific to Rails 3 or Rails 4. Gems in the latter two groups were mostly the same and only differed in version numbers. Another app I used this approach with was a small, but already messy Sinatra application where gemfiles helped me to extract some parts into gems.