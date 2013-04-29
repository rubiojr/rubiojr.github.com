---
layout: post
title: "Distributing complex ruby scripts with Omnibus generated Debian packages"
description: ""
category: hack
tags: [debian, packaging, ruby]
---
{% include JB/setup %}

Let's say I have a nice little ruby script I want to distribute to a 
large number of Debian/Ubuntu hosts. 

Ideally, all I have to do is to scp the script to the hosts 
and 'apt-get install ruby' on them, then I can execute my script.
Piece of cake.

I said ideally, because sometimes things get a little hairy if your script 
has a large number of rubygem deps or depends on a newer ruby version
not available in the repositories (hello! Ruby 2.0). Distributing and running 
the script on a remote server is not that easy anymore.

Consider the following script:

```ruby
# my-super-duper.rb
require 'fog' # bazillion deps
require 'sinatra' # even more deps


get '/foo' do
  # I do my Fog thingie here
  'I required fog and sinatra just to say HI'
end
```

Now, [Fog](https://github.com/fog/fog) and [Sinatra](http://sinatrarb.com) have been packaged (oh yeah!) and are available
in Ubuntu Precise repositories. Unfortunately, I need a newer Sinatra
version coz I want Feature X, available in version 1.4 and the packaged
version is 1.3.2 (shit).

Same thing with Fog, packaged version is 1.3.1 and I need 1.10!

What should I do? Some options to consider:

1. Roll up my sleeves and use Pupppet/Chef, spend 1/2/3 days to properly
   create and test the required manifests/cookbooks to the deploy the 
   whole thing. If you think that installing the script deps was cumbersome,
   get ready for this, because things aren't getting any easier with 
   Puppet/Chef.

2. I'm wheel re-inventor. My favorite thing is a Bash script to do the 
   work for me.
   Nothing wrong with the approach, perhaps old school and not
   so kewl anymore but been there, done that, and I still do it from time
   to time, so not a big deal.

3. Opscode omnibus-ruby to the rescue! 

   What if there is a magic tool out there that builds and bundles the ruby 
   ruby and ALL the dependencies in a single Debian package for you to install?
   That will definitely shorten the manifest/cookbook development cycle
   making it simpler easier to debug and test. 
   
   Even if you are part of the wheel re-inventors crew (like myself), the
   benefits would be great, since I only need a couple of of lines in the
   script to fetch and install the package and I don't have to care about
   deps, the ruby runtime and their versions. I don't even need them
   installed.

   See [http://github.com/opscode/omnibus-ruby](http://github.com/opscode/omnibus-ruby)

## Installing Omnibus

The idea is to create the package using Omnibus in your 
workstation/Vagrant box/chroot/lxc container, then distribute the binary 
package to the servers.

This guide assumes you are using Ubuntu 12.04. Things may vary slightly
in newer releases since the ruby landscape has changed (1.9.3 is
now the default ruby, IIRC).

You should run these commands as root, or use sudo. Even better, use a
clean environment (lxc, vagrant, chroot, etc) to follow the steps if
possible.

### Install the required packages

    apt-get install git ruby rubygems ruby1.9.3 build-essential ruby-dev

### Switch to ruby 1.9.3

Ubuntu Precise defaults to 1.8.7 and Omnibus requires 1.9 or later, so
let's change that:

    update-alternatives --set ruby /usr/bin/ruby1.9.1
    update-alternatives --set gem /usr/bin/gem1.9.1

Yeah, it's not a typo. /usr/bin/ruby1.9.1 is Ruby 1.9.3-p0 in Ubuntu 12.04.

### Install Omnibus and deps

Time to install Omnibus:

    gem install omnibus bundler --no-ri --no-rdoc

### Create the Omnibus project

Time to create the skeleton of the project that will help us to build
the fat package with our script included:

    omnibus project my-scripts
    cd omnibus-my-scripts
    bundle install --binstubs

### Add herbs and spices to the mix

Time to add our scripts and configurations and tell Omnibus which 
dependencies we want to include.

Edit the project configuration in **config/project/my-scripts.rb**. It should
look something like:

```ruby
name "my-scripts"
maintainer "Sergio Rubio <rubiojr@frameos.org>"
homepage "http://rubiojr.rbel.co"

replaces        "my-scripts"
install_path    "/opt/my-scripts"
build_version   "0.0.1"
build_iteration 1

# creates required build directories
dependency "preparation"

# my-scripts  dependencies/components
dependency "my-scripts"

# version manifest file
dependency "version-manifest"

exclude "\.git*"
exclude "bundler\/git"
```

Pay attention to the **dependency "my-scripts"**, we need to create the
**config/software/my-scripts.rb** file to define individual software 
components that go into making the package.

Create config/software/my-scripts.rb. It should look something like:

```ruby
name "my-scripts"

# I want to include all these deps plus ruby
# 
# The software files for these deps have been created for
# us already (see https://github.com/opscode/omnibus-software)
# and Omnibus will make use them if not found in our
# configuration.
#
dependency "libxml2"
dependency "libxslt"
dependency "ruby"
dependency "rubygems"
dependency "yajl"
dependency "bundler"

# my-super-duper.rb script gem deps. Will install latest versions
# available but that's OK for me
gem_deps = %w[fog colored sinatra]

# stuff I don't understand yet
relative_path "my-scripts"
always_build true

env = {
  "CFLAGS" => "-L#{install_dir}/embedded/lib -I#{install_dir}/embedded/include",
  "LDFLAGS" => "-Wl,-rpath #{install_dir}/embedded/lib -L#{install_dir}/embedded/lib -I#{install_dir}/embedded/include"
}

# My BIG hack to install all the scripts found in the scripts/
# directory of the project to /opt/my-scripts/bin along with the
# rest of the dependencies (ruby itself, gems, etc)
build do

  # Install required gem deps
  gem_deps.each do |g|
    gem "install #{g} -n #{install_dir}/bin --no-rdoc --no-ri", :env => env
  end

  FileUtils.mkdir_p "#{install_dir}/bin/"
  Dir["#{Omnibus::Config.project_root}/scripts/*"].each do |s|
    sname = File.basename(s)
    dest = "#{install_dir}/bin/#{sname}"
    File.open(dest, 'w') do |f|
      f.puts "#!#{install_dir}/embedded/bin/ruby"
      f.puts File.read(s)
    end
    FileUtils.chmod 0755, dest
  end
end
```

### Final step: add my-super-duper.rb to the package

Create the scripts directory in the project's directory and
add **my-super-duper.rb** script to it:
    
    [rubiojr.napoleon] pwd
    /home/rubiojr/Work/omnibus-my-scripts

    [rubiojr.napoleon] mkdir scripts/
    [rubiojr.napoleon] cp ~/Work/my-super-super.rb scripts/

The contents of the script included:

```ruby
# my-super-duper.rb
require 'fog' # bazillion deps
require 'sinatra' # even more deps


get '/foo' do
  # I do my complex Fog thingie here
  'I required fog and sinatra just to say HI'
end
```

### Building the Debian package

We're now ready to build the Omnibus Debian package:

    bin/omnibus build project my-scripts

If the build is successful, you should have a brand new Debian
package in the pkg/ directory with Ruby 1.9.3, your script and all 
the deps included. Time to distribute the package and do a bit of Yak Shaving.

    $ dpkg -i pkg/my-scripts_0.0.1-1.ubuntu.12.04_amd64.deb

    $ /opt/my-scripts/bin/my-super-duper.rb 
    [2013-04-29 20:55:13] INFO  WEBrick 1.3.1
    [2013-04-29 20:55:13] INFO  ruby 1.9.3 (2012-10-12) [x86_64-linux]
    == Sinatra/1.4.2 has taken the stage on 4567 for development with backup from WEBrick
    [2013-04-29 20:55:13] INFO  WEBrick::HTTPServer#start: pid=18220 port=4567

    $ curl http://localhost:4567/foo -q
    I required fog and sinatra just to say HI


Pretty cool and somewhat simple, isn't it?
